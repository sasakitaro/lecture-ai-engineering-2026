# 第6回 解答

## 演習
- STEP 1 のツールの名前：glossary_search_glossary
- STEP 2 の [tool call]：glossary_search_glossary({'query': '3レベル'})
- STEP 3 の [tool call]：fetch_fetch({'url': 'https://modelcontextprotocol.io/'}) ／ glossary_search_glossary({'query': 'MCP'})
- STEP 4 の [tool call]：aws_aws___search_documentation({'topics': ['general'], 'search_phrase': 'Amazon S3 versioning overview'})
- 気づき・疑問：
  - app/mcp_servers.json に書き足すだけで、エージェントのコードを変えずにツールが増えた（glossary → fetch → aws）。command があれば自分の環境で起動し、url があれば外部のサーバーに接続する。
  - STEP 3 では、fetch でページを読んだあと、モデルが自分で用語集の「MCP」も調べた。ツールが増えると、モデルが質問に合わせて組み合わせて使う。
  - STEP 4 のツール名は aws_aws___search_documentation。AWS 側のツール名（aws___search_documentation）の前に、設定の名前 aws_ が付くため、aws が重なっている。
  - aws は URL で接続するので、質問の内容が AWS に送られる。足す前に、提供元とリスクを確認する必要がある。

## 宿題
- 取り組んだレベル：実装（あわせて基礎の get_lecture_summary も公開）
- やったこと：
  - app/tools.py に、第3回で作った make_quiz を入れ、引数が範囲外のときに分かりやすいメッセージを返すようにした。
    - 1 未満、または用語集にある最大の回より大きいとき：「第N回は範囲外です。確認問題を作れるのは …です」
    - 範囲内でも用語がない回（第5回）：「第5回の用語は用語集にないため、確認問題を作れません」
    - どちらも JSON の error と available_lectures（作れる回の一覧）で返し、docstring に「error のときは問題を作らず、そのまま利用者に伝える」と書いた。
  - app/mcp_server.py で get_lecture_summary と make_quiz を公開した（エージェント側は変えていない）。
  - tests/test_tools.py に、範囲内（第3回）と範囲外（第0・5・99回）のテストを追加した。
- 結果：
  - uv run pytest -q：23 passed
  - --list-tools で glossary_get_lecture_summary と glossary_make_quiz が増えた（全体で 9 個）。
  - 「第99回のクイズを作って」の --trace：
    - [tool call]   glossary_make_quiz({'lecture': 99})
    - [tool return] glossary_make_quiz -> {"error": "第99回は範囲外です。確認問題を作れるのは 第1回、第2回、第3回、第4回、第6回 です。", "available_lectures": [1, 2, 3, 4, 6]}
    - [usage] 入力 5,770 tokens / 出力 72 tokens / モデル応答 2 回
    - モデルの回答：「第99回は講座の範囲外となります。確認問題を作れるのは以下の回です：第1回、第2回、第3回、第4回、第6回。どの回のクイズをご希望でしょうか？」
- 気づき・疑問：
  - ツールの返り値は、成功でもエラーでも、そのままモデルに渡る。エラーに「作れる回の一覧」を入れておくと、モデルが問題を勝手に作らず、代わりの回を示して聞き返した。
  - docstring に書いた「error のときはそのまま伝える」という指示どおりに、モデルが振る舞った。ツールの説明文は、呼ぶかどうかだけでなく、結果の扱い方も左右する。
  - ツールが 9 個になり、短い質問でも入力が 5,770 tokens になった。サーバーを足すほど、説明文の分だけ毎回の入力が増える。
  - fetch の説明文には「インターネットにアクセスできるようになった」という、モデルへの指示のような文が入っていた。他人のサーバーを足す前に、提供元だけでなく説明文も読む必要がある。
  - 疑問：エラーを文字列で返す場合と、例外（ModelRetry など）で返す場合とで、モデルの再試行のしかたがどう変わるか。
  