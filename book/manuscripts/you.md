---
class: content
---

<div class="doc-header">
  <h1>CodeCompanion.nvimでみるAIコーディングの実装</h1>
  <div class="doc-author">叶 恒志</div>
</div>

# CodeCompanion.nvimで見るAIコーディングの実装

## はじめに

本記事では Neovim の AI プラグインの CodeCompanion.nvim [^1]（以降 CodeCompanion）を軽く紹介した後、チャットの主要機能を実際の実装に基づいて詳細に解説していきます。

私はタブ補完の Copilot.lua [^2] をだけ使用してきました。しかし、コードを AI チャットクライアントにコピー&ペーストするのが面倒くさくなってきたし、
AI エージェントやコードコンテキストの理解を試したくなってきたので、CodeCompanion を導入しました。導入したはいいものの、エージェントの動作が少し不安定で、
いちいち共有したいファイルを選ぶのは思ったよりも面倒くさいです。少し手をいれる必要がありそうなので、そのための下準備として実装を理解しておきたいと思います。これに加え、AI コーディングが一般的に何をしているかを把握しておきたいという思いもあります。

## CodeCompanion.nvimとは

CodeCompanion は Neovim で AI コーディングを実現するプラグインの一つです。他には、Avante.nvim [^3] が多く使われいて、思想の違い（Zed AI [^4] ライクか Cursor [^5] ライクか）はあれど、共通して開いているファイルなどのコンテキスト共有機能、ファイル変更やコマンド実行を行う AI エージェント機能を提供しています。CodeCompanion は個別の処理に別々のモデルを選択できるのが面白い所ですが、今回は割愛します。また、**チャットに関連する機能**に絞って、実装を見ていきます。

<figure>
  <img src="images_you/companion_chat.png" alt="CodeCompanionのチャット画面" width="500">
  <figcaption>CodeCompanionのチャットインターフェース画面</figcaption>
</figure>

あいさつすると決まった概要の説明をしてくれます。

## CodeCompanionの使い方

CodeCompanion の基本的な使い方をいくつか紹介します。私は普段、GitHub Copilot [^6] で **Claude-3.7-sonnet モデル**を使用しています。

CodeCompanion ではチャットを打つ際に、読み込みたいバッファや外部コンテキストの参照、AI に実行してほしいツールの指定をプロンプトに含めることができます。

| 分類 | 記号 | 概要 |
| --- | --- | --- |
| 変数 | `#` | バッファの参照や LSP の診断情報を変数として文に書ける |
| スラッシュコマンド | `/` | 特定のプロンプトやアクションを実行する（ファイルを選んで、参照に追加） |
| ツール | `@` | LLM に実行してほしいツール（MCP、CMD）を共有できる |

## #buffer

#buffer を含めて書くことで，カレントバッファの内容を共有することができます

```
#bufferを説明してください
```

カレントバッファ（積分をする簡単な C のコード）について説明しています。

<figure>
  <img src="images_you/buffer.png" alt="バッファに関する説明" width="350">
  <figcaption>#buffer でカレントバッファを共有</figcaption>
</figure>

### /fetch

/fetch で共有したい URL を指定することができます。

試しに CodeCompanion の GitHub を共有して聞いてみます。

/fetch を入力すると、URL の入力を求められます。

<figure>
  <img src="images_you/fetch.png" alt="fetchコマンドの入力" width="350">
  <figcaption>URL を入力</figcaption>
</figure>

fetch に成功すると UI に `Sharing:` と表示され URL の参照が追加されたとわかります。回答も URL の内容に基づいています。

<figure>
  <img src="images_you/fetch_result.png" alt="fetchコマンドの結果" width="350">
  <figcaption>/fetch で URL を共有</figcaption>
</figure>

### @editor

@editor を書くと、LLM は Neovim バッファ内のコードを修正してくれるようになります。

バッファを指定し、@editor を含めて、指示してみます。
<figure>
  <img src="images_you/editor.png" alt="editorツールの使用" width="350">
  <figcaption>@editor でファイル編集を指示</figcaption>
</figure>

レスポンスが返ってくると、diff が表示され、accept もしくは reject ができます。

なお、~~@editor を記述しても、たまにファイル修正が行われません......。~~ 最新のバージョンでは動作は安定しています。

<figure>
  <img src="images_you/editor_result.png" alt="diffの表示" width="350">
  <figcaption>diff画面</figcaption>
</figure>

### チャット送信の実装

チャット機能に関する実装を見ていきます。

```
lua/
├── codecompanion/
│   ├── actions/
│   ├── adapters/
│   ├── helpers/
│   ├── providers/
│   ├── strategies/
│   │   ├── chat/
│   │   │   ├── agents/
│   │   │   ├── slash_commands/
│   │   │   ├── variables/
│   │   │   ├── debug.lua
│   │   │   ├── helpers.lua
│   │   │   ├── init.lua
│   │   │   ├── keymaps.lua
│   │   │   ├── references.lua
│   │   │   ├── subscribers.lua
│   │   │   ├── ui.lua
│   │   │   └── watchers.lua
```

LLM のモデルの操作には `adapters`、ツールの実行には `agents` でインタフェースを用意している、保守性の高い設計です。チャット機能の実装は `lua/strategies/chat/` にあり、`init.lua` がエントリーファイルで、全体の処理を制御します。

`chat/init.lua` に `Chat` クラスがありデータと各操作が定義されています。

まず、チャットバッファで管理するデータについて確認しておきたいと思います。

```lua
function Chat.new(args)
    local id = math.random(10000000)
    log:trace("Chat created with ID %d", id)
    local self = setmetatable({
        context = args.context,
        cycle = 1,
        header_line = 1,
        from_prompt_library = args.from_prompt_library or false,
        id = id,
        last_role = args.last_role or config.constants.USER_ROLE,
        messages = args.messages or {},
        opts = args,
        refs = {},
        status = "",
        tool_flags = {},
        tools_in_use = {},
        create_buf = function()
            local bufnr = api.nvim_create_buf(false, true)
            api.nvim_buf_set_name(bufnr, string.format("[CodeCompanion] %d", id))
            vim.bo[bufnr].filetype = "codecompanion"
            return bufnr
        end,
    }, { __index = Chat }}
```

チャット履歴を `messages`、やり取りしているターン数を `cycle`、最新のチャットの先頭行数を `header_line` に保持しています。

基本ユーザのチャット送信にトリガして処理が走るので、`function Chat:submit(opts)` を追っていきます。
`function Chat:submit(opts)` のフローはバリデーションや UI 制御などを除くと、以下のようになります。

```lua
Chat:submit開始
  ↓
1. チャットバッファからメッセージ解析 ts_parse_messages(self, self.header_line)
  ↓
2. ウォッチャーによる変更チェック ---UIで参照先の変更検知を読み込む
  ↓
3. messagesテーブルにメッセージ追加 self.addmessage()
  ↓
4. 変数とツール参照の置換     apply_tools_and_variables(message)
  ↓
5. 参照チェックとピン追加 ---UIでの参照の追加削除、固定操作を読み込む
  ↓
6. アダプター設定処理 ---アダプターは LLM とやり取りするためのインタフェースです
  ↓
7. HTTPリクエスト送信  clinet.new():request
  ↓
8. コールバックでのレスポンス処理     
				self.current_request = client
	        .new({ adapter = settings })
	        :request(self.adapter:map_roles(vim.deepcopy(self.messages)),
  ↓
9. done処理（コールバックの完了時)
```

まず、先頭行番号から最新のチャットを切り取ります。チャットはマークダウンで書かれるので、空欄や修飾子を除いて新しいメッセージを作成します。

```lua
{
  message = {
    content = "#bufferを説明して"
  }
}
```

これを `messages` テーブルに追加します。その後は変数やツールの修飾子(#@)を読み取り、対応する処理を行います。

```lua
function Chat:apply_tools_and_variables(message)
    if self.agents:parse(self, message) then
        message.content = self.agents:replace(message.content)
    end
    if self.variables:parse(self, message) then
        message.content = self.variables:replace(message.content, self.context.bufnr)
    end
end
```

`message` をパースして、#buffer や @editor を検出して、機能を実行します。その後は参照が分かるように #buffer などのシンボルを入れ替えます。(例: #buffer→`buffer 7 ('server.ts')`)

```lua
=== messages ===
{ {
    content = "You are an AI programming assistant named \"CodeCompanion\". You are currently plugged into the Neovim text editor on a user's machine.\n\nYour core tasks include:\n- Answering general programming questions.\n- Explaining how the code in a Neovim buffer works.\n- Reviewing the selected code in a Neovim buffer.\n- Generating unit tests for the selected code.\n- Proposing fixes for problems in the selected code.省略",
    cycle = 1,
    id = 1878391891,
    opts = {
      tag = "from_config",
      visible = false
    },
    role = "system"
  }, {
    content = "hello",
    cycle = 1,
    id = 1509314226,
    opts = {
      visible = true
    },
    role = "user"
  }
} }
```

新しいチャット内容を、元のシステムプロンプトに追加して LLM に送ります。直前の 5 件のメッセージまで送信といった制限はなく、**バッファ内のすべてやり取りを送信**する仕様です。各 LLM にトークン数の上限を設定でき（default:15000）、上限を超えるとエラーとなり、リクエストが送信されません。

```lua
  if has_tools then
    tools = self.adapter.handlers.tools.format_tool_calls(self.adapter, tools)
    self:add_message({
      role = config.constants.LLM_ROLE,
      tool_calls = tools,
      opts = {
        visible = false,
      },
    })
    return self.agents:execute(self, tools)
  end
```

最後にコールバックでレスポンスを UI（チャットバッファ）に書き込み、LLM の返答からツールを実行します。

## 各機能の動作

各機能に応じた処理を見ていきます。それらの処理は共通して、チャットバッファのメッセージをパースして、対応するモジュールを呼び出して、`messages` にメッセージを追加します。

```lua
local ok, module = pcall(require, "codecompanion." .. path .. "." .. variable)
```

設定ファイルに書いたネームスペースから解決します。

```lua
    --config.lua
    ["buffer"] = {
      callback = "strategies.chat.variables.buffer",
      description = "Share the current buffer with the LLM",
      opts = {
        contains_code = true,
        has_params = true,
      },
```

自作した機能やプロンプトを設定ファイルに書くことで簡単に使用できるようになるので、拡張性はかなり高いです。

### スラッシュコマンド（/）によるコンテキスト共有

まず、スラッシュコマンドの入力から見ていきます。

```lua
api.nvim_create_autocmd("CompleteDone", {
    group = self.aug,
    buffer = bufnr,
    callback = function()
        local item = vim.v.completed_item
        if item.user_data and item.user_data.type == "slash_command" then
            -- Clear the word from the buffer
            local row, col = unpack(api.nvim_win_get_cursor(0))
            api.nvim_buf_set_text(bufnr, row - 1, col - #item.word, row - 1, col, { "" })

            completion.slash_commands_execute(item.user_data, self)
        end
    end,
})
```

どうやら、`CompleteDone`（入力の補完完了）にトリガして、実行しているようです。補完系のプラグインを導入していない環境や動作を独自に変えている場合はうまく動作しないかもしれません。私の環境ではプラグイン **nvim-cmp** を外したら動作しません。

/fetch コマンドの動作を見ていきます。

/fetch は `lua/codecompanion/Chat/slash_commands/fetch.lua` にあります。

パースには jina.ai [^6] を使用しています。jina.ai は URL やウェブ検索から LLM に適した入力を得られる API で、テキストをのみを抽出しています。PDF を読み取れる点と画像の注を抽出できる点で便利そうです。ただテキストを抽出しているだけなので、トークン数に気を配る必要がありそうです。

```lua
 {
    content = "Here is the output from https://news.yahoo.co.jp/pickup/6537567 that I'm sharing with you:\n\n<content>\nYahoo!ニュース\n\n最短3分から受験可能！　ヤフー防災模試で備えを\n\nYahoo! JAPANヘルプ\nキーワード：\n 検索\n須賀健太\n\t\nIDでもっと便利に新規取得\nログイン アプリはじめて利用限定、お得な半額クーポン\nマイページ\n購入履歴\nトップ\n速報\nライブ\nエキスパート\nオリジナル\nみんなの意見\nランキング\n有料\n主要\n国内\n国際\n経済\nエンタメ\nスポーツ\nIT\n科学\nライフ\n地域\nトピックス一覧\n首相 省略\n© LY Corporation\n</content>",
    cycle = 1,
    id = 1972871335,
    opts = {
      reference = "<url>https://news.yahoo.co.jp/pickup/6537567</url>",
      visible = false
    },
    role = "user"
   }
```

抽出したテキストにプロンプト（`Here is the output from`）を加えて、メッセージに追加しています。

```lua
  { content = "Here is the output from https://platform.openai.com/docs/guides/function-calling?api-mode=chat&lang=curl&example=get-weather#overview that I'm sharing with you:\n\n<content>\n\n</content>",
    cycle = 1,
    id = 421072586,
    opts = {
      reference = "<url>https://platform.openai.com/docs/guides/function-calling?api-mode=chat&lang=curl&example=get-weather#overview</url>",
      visible = false
    },
    role = "user"
  } }
```

サイト（open.ai）によっては、Bot 対策により内容を抽出できないので、その場合は **URL のみ**をメッセージに追加します。しかし、テキストを含めない場合でも、返答は少し異なれど、精度の差はほとんどない印象でした（**GitHub Copilot claude-3.7-sonnet** の場合）。テキスト抽出をせずに、URL だけをメッセージに追加したほうが、トークン数的にはいいのかもしれません。

スラッシュコマンドによるコンテキスト共有は、変数やツールと `messages` への追加のタイミングが異なります。スラッシュコマンドはチャットを編集している途中、変数やツールは送信するタイミングで追加されます。これにより、コンテキストを送ってから指示をするのか、指示してからコンテキストを送るかで、ほんの少し LLM へのお願いの仕方が異なります。返答の質にほとんど影響がないと思いますが、もしかすると役に立つかもしれないので、メモしておきます。

### 変数（#）によるコンテキスト共有
#buffer を例に動作を見ていきます。

```lua
  local content = buf_utils.format_with_line_numbers(bufnr)
```

`format_with_line_numbers()` はバッファの参照を受取り，コードの行番号を毎行付与して、空白を削除します。

```lua
 {
    content = "buffer 7 (`server.ts`)を説明して",
    cycle = 1,
    id = 1509314226,
    opts = {
      visible = true
    },
    role = "user"
  }, {
    content = 'Here is the content from the buffer.\n\nBuffer Number: 7\nName: server.ts\nPath: /Users/koshi/koshi_workspace/comicEmotion/src/server.ts\nFiletype: typescript\nContent:**コードは省略します**\n',
    cycle = 1,
    id = 197956778,
    opts = {
      reference = "<buf>server.ts</buf>",
      tag = "variable",
      visible = false
    },
    role = "user"
```

（#buffer→`buffer 7 ('server.ts')`）、チャットの変数を入れ替えて、LLM に参照がわかるようにバッファ情報をコード本体に付けて、`messages` に加えます。

#buffer では指定したバッファの内容をそのまま LLM に送ります。ファイルの読み込みも同様に、行番号付けてそのまま送ります。また、CodeCompanion にはプロジェクト下のファイルをすべて読み込むというような機能はありません。プロジェクト単位で JSON ファイルで読み込みたいファイルを設定できますが、ファイルを個別に一つずつ書く必要があって非常に面倒です。一方で、意図せずにライブラリのコードすべて送信するようなことは起こりにくいです。

### ツール（@）によるツール実行

CodeCompanion のツール実行は、**ツールの意図**や**スキーマを規定したプロンプト**にを送ることで、LLM にスキーマに従った返答を要求します。返答に含まれるスキーマをプラグイン側で解釈して、対応する操作を実行します。これは一般的に "Tool use" と呼ばれる AI エージェントのデザインパターン
の実装です。

<figure>
  <img src="images_you/architecture.png" alt="ツール実行のアーキテクチャ" width="350">
  <figcaption>ツール実行のシーケンス図</figcaption>
</figure>

ツール実行のアーキテクチャです。

- **Agent**: LLM のレスポンスからツール指示を解析し、実行を管理
- **Tool Executor**: ツールの実行を担当
- **Tool**: 実際の機能を実装したモジュール（@editor など）

Agent で LLM からの指示をパースして、キューに積んでいきます。ToolExecutor はキューが空になるまで、合致したツールを実行します。

このアーキテクチャでは**エージェントは LLM からの指示を一括で実行します**。

> 
> LLM 「ファイル作って」　Agent「作った」
> 
> LLM「おけ　次はファイル内容を書き換えて」　Agent「おわった」
> 
> LLM「おけ　次は…」

ではなく

> 
> LLM 「ファイル作ってからこの内容にして」　Agent「わかりました」

というようなやり取りをします。柔軟さに欠けるので複雑なタスクには向かないかもしれません。

@editor を例に具体的な動作を見ていきます。

`apply_tools_and_variables(message)` で `agents:parse()` を呼び出し、メッセージに含まれる @editor を検出して、対応するツールを実行します。

@editor の場合はまず、ツール仕様全般とツールのスキーマについて指示書、そしてコードを `messages` に加えます。英語のプロンプトですが、見やすさのため日本語訳しています。

```markdown
# ツールアクセスと実行ガイドライン

### 概要
特定のタスクをサポートするための専門ツールが利用可能になりました。これらのツールはユーザが明示的に要求した場合にのみ使用できます。

### 一般的なルール
- **ユーザー主導の実行:** ユーザーが特定のツールの使用を明示的に指示した場合のみ使用する（例：cmd_runner の場合は「コマンドを実行して」など）
- **厳格なスキーマ準拠:** ツール呼び出し時には提供された XML スキーマに正確に従う
- **XML 形式:** 応答は常に XML として指定されたマークダウンコードブロック内で、`<tools></tools>` タグで囲む
- **有効な XML:** 構築する XML は有効で整形式である必要がある
- **複数コマンド:**
  - 同じタイプのコマンドを発行する場合は、一つの `<tools></tools>` ブロック内に複数の `<action></action>` エントリを含める
  - 異なるツールのコマンドを発行する場合は、それぞれを `<tool></tool>` タグで囲み、`<tools></tools>` ブロック内に配置する
- **副作用なし:** ツールの呼び出しは基本タスクや会話構造を変更するべきではない
```
明示的に指示した場合のみ使用するとありますが、指示してない場合でも、ツールが実行されることがあります......。
CodeCompanion では独自に `XML` の出力を定義して、LLM に出力されるアプローチを取ってい~~ます。~~ました。

執筆している途中で、CodeCompanion の新バージョンがリリースされ、方式が変更されました。今は、OpenAI が定めた、LLM がアプリケーションでツール呼び出し枠組みである Function calling [^7] のスキーマ（JSON）に合わせるように変更されています。当初は、まだ Function calling の実装が進んでいなく、独自のスキーマを考案する必要があったが、現在は多くの LLM は Function calling に対してファインチューニングしているので、精度が高いとのこと。

> @editor を記述しても、たまにファイル修正が行われません......。
> 

これは `XML` 形式の指示の精度によるもので、Function calling では安定しています。

```markdown
# エディターツール (`editor`) - 拡張ガイドライン
### 目的:　省略
### 使用タイミング:　省略
### 実行形式:　省略
- 常に XML マークダウンコードブロックを返す
- ユーザーが共有したバッファ番号を必ず `<buffer></buffer>` タグに含める（提供されていない場合はユーザーに確認）
- 各コード操作は以下の条件を満たす必要がある:
  - 特殊文字を保持するため CDATA セクションで囲む（CDATA セクションにより `<` や `&` などの文字が XML マークアップとして解釈されない）
  - XML スキーマに厳密に従う
- 複数のアクション（追加、更新、削除）を順番に実行する場合は、一つの XML ブロック内の `<tool></tool>` タグ内に、個別の `<action></action>` エントリとして組み合わせる
### XML スキーマ:　省略
ツール呼び出しは以下の構造に従う必要があります:
d) **複数アクション**（複数のアクションを順番に実行する必要がある場合）:
xml
<tools>
  <tool name="editor">
    <action type="delete">
      <end_line>13</end_line>
      <start_line>13</start_line>
      <buffer>5</buffer>
    </action>
    <action type="add">
      <line>20</line>
      <buffer>5</buffer>
      <code><![CDATA[function M.hello_world()]]></code>
    </action>
  </tool>
</tools>
### 重要な考慮事項:　省略
### 注意事項:　省略
```
ファイル編集の XML の例をプロンプトに含んでいて、行数とバッファ番号で編集箇所を返すように指示します。

```xml

<tools>
  <tool name="editor">
    <action type="update">
      <buffer>51</buffer>
      <start_line>7</start_line>
      <end_line>7</end_line>
      <code><![CDATA[double integral (double left, double right, int num){]]></code>
    </action>
    <action type="update">
      <buffer>51</buffer>
      <start_line>28</start_line>
      <end_line>37</end_line>
      <code><![CDATA[int main (void) {
    int left, right, num;
    printf ("積分範囲を入力してください\n");
    printf ("左: ");
    scanf ("%d", &left);
    printf ("右: ");
    scanf ("%d", &right);
    printf ("分割数: ");
    scanf ("%d", &num);
    printf ("∫[%d,%d] x² dx = %f\n", left, right, integral(left, right, num));
    return 0;
}
]]></code>
    </action>
  </tool>
</tools>
```

このような `XML` が返ってくるので、これをパースして、適したツールを実行します。

## まとめ

CodeCompanion の各機能の処理の流れや、コンテキスト共有の内容を確認しました。

- /fetch は web ページのテキスト丸ごと
- /buffer はファイルのテキスト丸ごと
- @editor は LLM に XML 出力を指示して、返ってきた XML を読み取り実行

実装を見る前は、コンテキストの読み込みや AI によるツール実行は魔法みたいなことをしていると思ってたので、率直な実装で拍子抜けした印象です。そりゃ、変に削らず、丸ごと送るほうが精度がいいだろ！API を直接叩いている場合は、トークン数を節約したいので、なんらかの工夫はしたいですが。CodeCompanion の一つ一つの機能はシンプルで、簡単に機能を追加、組み合わせてもコンフリクトが起こりづらく、拡張性に優れている設計が特徴で、改良して長く使っていけるプラグインだと感じました。

CodeCompanion を導入した素の状態では、コンテキスト読み込みの機能が単純なものしかなく、Cursor のように賢くやってくれません。Avante.nvim のように、プロジェクトのファイルをインデックス化して、読み込む機能（Codebase）はありません。また、バッファやファイルの読み込みは自分で明示的に書く必要があります。私はエディタ内で AI チャットを使うからには、現在のプロジェクトや作業ディレクトリを勝手に読み込んでいてほしい派なので、今後はこの実装を踏まえて、自分にあったコンテキスト読み込みを実現していきたいと思います。

拡張機能として、プロジェクトをインデックス化する VectorCode [^8] が既にあり、連携させることで Codebase を実現することができます。ただし、VectorCode の CLI を使って、予めプロジェクトをインデックス化する必要があります。操作が面倒で、関心が分散しているので、扱いにくさを感じます。Cursor のように、暗黙的に読み込ませるにはある程度コードを理解して、自分でコードを書く努力が必要です。自分のワークフローに改造できるのが Neovim の良さではありますが、逆に改造しないと、主流のエディターに利便性で劣るので、Neovim を使っていくためには熱が必要とつくづく思いました。


[^1]: CodeCompanion.nvim: GitHub: https://github.com/Bryley/codecompanion.nvim 

[^2]: Copilot.lua: GitHub: https://github.com/zbirenbaum/copilot.lua

[^3]: Avante.nvim: GitHub: https://github.com/willothy/avante.nvim

[^4]: Zed AI: Zed エディターに組み込まれた AI コーディング機能。https://zed.dev/

[^5]: Cursor: https://cursor.sh/

[^6]: GitHub Copilot: https://github.com/features/copilot

[^7]: Function calling: https://platform.openai.com/docs/guides/function-calling

[^8]: VectorCode: https://github.com/Davidyz/VectorCode
