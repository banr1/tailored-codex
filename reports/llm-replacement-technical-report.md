# 技術レポート: Codex CLI における LLM モデル置き換えの技術的可能性

## エグゼクティブサマリー

**結論: 技術的に可能です。ただし、使用するAPIモードによって機能の制約があります。**

Codex CLIは2つのWire APIプロトコルをサポートしており、**Chat Completions API**モードを使用すれば、OpenAI互換インターフェースを持つ多くのLLMサービス（Ollama、LM Studio、vLLM、LocalAI、Azure OpenAI、Anthropic Claude経由のプロキシ等）に置き換えることが可能です。

---

## 1. アーキテクチャ概要

### 1.1 サポートされるWire API

Codex CLIは以下の2つのAPIプロトコルをサポートしています：

| API | エンドポイント | 状態 | 互換性 |
|-----|---------------|------|--------|
| **Responses API** | `/v1/responses` | メイン（推奨） | OpenAI固有 |
| **Chat Completions API** | `/v1/chat/completions` | フォールバック（deprecated） | OpenAI互換多数 |

**重要**: `wire_api = "chat"` を設定すれば、Chat Completions API互換のサービスに接続可能です。

### 1.2 主要実装ファイル

```
codex-rs/
├── core/src/
│   ├── model_provider_info.rs   # プロバイダー設定定義
│   ├── client.rs                # メインクライアント
│   └── api_bridge.rs            # エラー処理
├── codex-api/src/
│   ├── provider.rs              # Provider構造体
│   ├── endpoint/chat.rs         # Chat Completions API実装
│   ├── requests/chat.rs         # リクエストビルダー
│   └── sse/chat.rs              # SSEストリーミング処理
```

---

## 2. 置き換え方法

### 2.1 設定ファイルによるカスタムプロバイダー追加

`~/.codex/config.toml` に以下のようにカスタムプロバイダーを追加：

```toml
# 使用するプロバイダーを指定
model_provider = "my_custom_provider"
model = "my-model-name"

# カスタムプロバイダー定義
[model_providers.my_custom_provider]
name = "My Custom LLM"
base_url = "http://localhost:8080/v1"
wire_api = "chat"                        # Chat Completions API を使用
env_key = "MY_API_KEY"                   # API Keyの環境変数（オプション）
```

### 2.2 組み込みOSSプロバイダー

既に以下のプロバイダーが組み込まれています：

| Provider ID | デフォルトポート | Wire API |
|-------------|-----------------|----------|
| `ollama` | 11434 | responses |
| `ollama-chat` | 11434 | **chat** |
| `lmstudio` | 1234 | responses |

**使用例（Ollama Chat API）**:
```bash
export CODEX_OSS_BASE_URL="http://localhost:11434/v1"
codex -c model_provider=ollama-chat -c model=llama3
```

### 2.3 環境変数によるオーバーライド

```bash
# OpenAIプロバイダーのbase_urlを上書き
export OPENAI_BASE_URL="https://my-proxy.example.com/v1"

# OSSプロバイダー用
export CODEX_OSS_BASE_URL="http://localhost:8080/v1"
export CODEX_OSS_PORT="8080"
```

---

## 3. Chat Completions API 互換性詳細

### 3.1 リクエスト形式

Codex CLIが生成するChat Completions APIリクエスト（`codex-rs/codex-api/src/requests/chat.rs:294-299`）：

```json
{
  "model": "model-name",
  "messages": [
    {"role": "system", "content": "instructions"},
    {"role": "user", "content": "user input"},
    {"role": "assistant", "content": "...", "tool_calls": [...]}
  ],
  "stream": true,
  "tools": [...]
}
```

### 3.2 レスポンス形式（SSE）

期待されるストリーミングレスポンス形式（`codex-rs/codex-api/src/sse/chat.rs`）：

```
data: {"choices":[{"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"choices":[{"delta":{"tool_calls":[{"id":"call_1","function":{"name":"read_file","arguments":"{\"path\":\"a.txt\"}"}}]},"finish_reason":"tool_calls"}]}

data: [DONE]
```

### 3.3 必須サポート機能

互換LLMサービスが対応すべき機能：

| 機能 | 必須度 | 備考 |
|------|--------|------|
| `messages` 配列 | **必須** | system/user/assistant/tool ロール |
| `stream: true` | **必須** | SSEストリーミング |
| `tools` | 推奨 | Function Calling サポート |
| `tool_calls` | 推奨 | ツール呼び出しレスポンス |
| `finish_reason` | 推奨 | "stop" / "tool_calls" / "length" |

---

## 4. 機能制約と互換性マトリクス

### 4.1 APIモード別機能比較

| 機能 | Responses API | Chat Completions API |
|------|---------------|---------------------|
| 基本的な会話 | ✅ | ✅ |
| Function Calling | ✅ | ✅ |
| ストリーミング | ✅（SSE） | ✅（SSE） |
| 推論詳細（reasoning） | ✅ | ⚠️ 部分的 |
| 推論サマリー | ✅ | ❌ |
| text.verbosity | ✅ | ❌ |
| 並列ツール呼び出し | ✅ | ⚠️ プロバイダー依存 |
| prompt_cache_key | ✅ | ❌ |
| conversation_id | ✅ | ⚠️ ヘッダーのみ |
| WebSocket接続 | ✅ | ❌ |

### 4.2 互換LLMサービス対応状況（推定）

| サービス | Chat API互換 | Function Calling | 備考 |
|----------|-------------|------------------|------|
| **Ollama** | ✅ | ✅ | 組み込みサポート |
| **LM Studio** | ✅ | ✅ | 組み込みサポート |
| **vLLM** | ✅ | ✅ | OpenAI互換モード |
| **LocalAI** | ✅ | ✅ | 完全互換 |
| **Anthropic Claude** | ❌ | - | プロキシ必要 |
| **Azure OpenAI** | ✅ | ✅ | query_params設定必要 |
| **Together AI** | ✅ | ✅ | base_url変更のみ |
| **Groq** | ✅ | ✅ | base_url変更のみ |
| **Fireworks AI** | ✅ | ✅ | base_url変更のみ |

---

## 5. 実装上の注意点

### 5.1 deprecation警告

`codex-rs/core/src/model_provider_info.rs:30`:
```rust
pub const CHAT_WIRE_API_DEPRECATION_SUMMARY: &str =
    r#"Support for the "chat" wire API is deprecated and will soon be removed..."#;
```

**影響**: 将来のバージョンでChat APIサポートが削除される可能性があります。

### 5.2 ツール呼び出しIDの生成

Chat Completions APIでは、ツール呼び出しIDがない場合、Codex側で生成されます（`sse/chat.rs:311`）：
```rust
call_id: id.unwrap_or_else(|| format!("tool-call-{index}"))
```

### 5.3 認証の差異

```rust
// model_provider_info.rs より
pub struct ModelProviderInfo {
    pub env_key: Option<String>,                    // 環境変数からAPIキー
    pub experimental_bearer_token: Option<String>,  // 直接指定（非推奨）
    pub requires_openai_auth: bool,                 // OpenAI認証フローの要否
}
```

---

## 6. 推奨設定例

### 6.1 Ollama + Llama 3

```toml
model_provider = "ollama-chat"
model = "llama3:70b"

[model_providers.ollama-chat]
name = "Ollama"
base_url = "http://localhost:11434/v1"
wire_api = "chat"
```

### 6.2 vLLM サーバー

```toml
model_provider = "vllm"
model = "meta-llama/Llama-3-70b-chat-hf"

[model_providers.vllm]
name = "vLLM"
base_url = "http://localhost:8000/v1"
wire_api = "chat"
```

### 6.3 Azure OpenAI

```toml
model_provider = "azure"
model = "gpt-4o"

[model_providers.azure]
name = "Azure OpenAI"
base_url = "https://YOUR_RESOURCE.openai.azure.com/openai/deployments/YOUR_DEPLOYMENT"
wire_api = "chat"
env_key = "AZURE_OPENAI_API_KEY"
query_params = { "api-version" = "2024-02-15-preview" }
```

---

## 7. 結論と推奨事項

### 7.1 実現可能性評価

| 観点 | 評価 | 説明 |
|------|------|------|
| **技術的実現可能性** | ✅ 高い | Chat Completions APIモードで対応可能 |
| **設定の容易さ** | ✅ 高い | config.toml編集のみ |
| **機能の完全性** | ⚠️ 中程度 | 一部の高度な機能は利用不可 |
| **将来の持続性** | ⚠️ 低い | Chat APIはdeprecated |

### 7.2 推奨アプローチ

1. **短期的**: `wire_api = "chat"` を使用してChat Completions API互換サービスに接続
2. **中期的**: Responses API対応のラッパー/プロキシサーバーを構築
3. **長期的**: Codex CLIのフォークまたはResponses API実装の監視

### 7.3 リスク

- Chat APIサポートの将来的な削除
- 高度な機能（推論詳細、verbosity制御等）が利用不可
- プロバイダー固有の挙動の差異

---

## 8. 検証手順

置き換えが正しく動作することを確認するための手順：

1. カスタムプロバイダー設定を `~/.codex/config.toml` に追加
2. LLMサービスを起動（例: `ollama serve`）
3. 以下のコマンドでテスト：
   ```bash
   cd codex-rs
   cargo run --bin codex -- -c model_provider=ollama-chat -c model=llama3 "Hello, World!"
   ```
4. ツール呼び出しのテスト：
   ```bash
   cargo run --bin codex -- -c model_provider=ollama-chat -c model=llama3 "List files in the current directory"
   ```

---

## 付録: 調査対象ファイル一覧

### コア実装
- `codex-rs/core/src/model_provider_info.rs` - プロバイダー設定・組み込みプロバイダー定義
- `codex-rs/core/src/client.rs` - メインクライアント実装
- `codex-rs/core/src/api_bridge.rs` - エラーマッピング・認証ブリッジ
- `codex-rs/core/src/default_client.rs` - デフォルトHTTPクライアント構築

### API層
- `codex-rs/codex-api/src/provider.rs` - Provider構造体・WireApi enum
- `codex-rs/codex-api/src/endpoint/chat.rs` - ChatClient実装
- `codex-rs/codex-api/src/requests/chat.rs` - Chat APIリクエストビルダー
- `codex-rs/codex-api/src/sse/chat.rs` - Chat API SSEストリーミング処理

### プロトコル定義
- `codex-rs/protocol/src/models.rs` - ResponseItem等のデータモデル
- `codex-rs/protocol/src/openai_models.rs` - OpenAI互換モデル定義

---

**作成日**: 2026-01-16
**調査対象**: Codex CLI (codex-rs/)
**調査者**: Claude
