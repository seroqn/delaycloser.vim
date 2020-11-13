# tozzy.vim
Insert-mode にて、クォートや括弧など（ペア）の、閉じ文字を補完します。

既存のペア補完系プラグインが開き文字（例えば `{` ）を入力したとき直後に即座に閉じ文字 `}` を補完するのに対して、当プラグインは開き文字の後に任意の一文字が入力されるまで補完を遅延します（ `{` では閉じ文字は補完されず `{x` までタイプしたときはじめて `}` が補完されます）。

マルチバイト文字に対応します。割と古い Vim でも動くかもしれません。

某プラグインと違って閉じ文字の前で閉じ文字を入力しても閉じ文字を脱出することはできません（例外あり）。補完された閉じ文字を抜けるときは基本は `<Left>` か、または `tozzy#leave()` をキーマップに割り当ててご利用ください。

判定を遅延させることで多少は誤爆しにくくなったと思いますが、それでも若干の誤爆は生じます。

誤爆例
- 入れ子になった空の括弧を並べたいとき `[[], []]`、内側の括弧を閉じた入力の時点で一番外側に出てしまう。
- ある単語をクォートで囲いたいとき、閉じるクォートを入力した後の入力でクォート開始と判定されてしまう（サラウンド系の他のプラグインなどで解決可能）。

## 設定
### キーマッピング
なるべくキーマップを汚さないように心がけていましたが、`i_<C-R>` で挿入された文字列と通常にタイプした文字列を判別できない不具合が見つかったので、`i_<C-R>` は事前に `tozzy#i_ctrl_r_alt()` でマッピング設定します。

#### `i_<C-R>`の代替― `tozzy#i_ctrl_r_alt()`
```vim
inoremap <silent><expr><C-r> tozzy#i_ctrl_r_alt()
```
インサートモードにて、`i_ctrl-R` の役割を果たします。`i_ctrl-R` で挿入された文字と通常にタイプされた文字を当プラグインに区別させるために使います。

`tozzy#i_ctrl_r_alt()` は `"\<C-R>"` を返すので通常の `"\<C-R>"` の代わりに使えます。

もしも次のような、インサートモードでレジスタを挿入するための代替マップを .vimrc で定義していた場合、
```vim
inoremap <C-r><C-e>   <C-r>"
inoremap <C-r><C-y>   <C-r>+
inoremap <C-r><C-f>   <C-r>=expand('%:t')<CR>
```

代わりに以下のように書き換えることができます。`tozzy#i_ctrl_r_alt()` に引数として渡された辞書はキーに`<C-r>` に続く2番目のキーマップ、値に `<C-r>` に続くリマップを定義します。
```vim
inoremap <silent><expr><C-r> tozzy#i_ctrl_r_alt({"\<C-e>": '"', "\<C-y>": "+", "\<C-f>": "\<C-r>=expand('%:t')\<CR>"})
```

#### 閉じ文字を抜ける― `tozzy#is_leavable()` と `tozzy#leave()`

`((x` と入力して、`((x|))` となっているとき（`|` はカーソル）、`tozzy#leave()` を（`:map-<expr>` モードで）実行すると、閉じ括弧を抜けて `((x))|` にカーソルが移動します。`tozzy#leave()` が実行できるとき `tozzy#is_leavable()` は非0 になります。

例えばタブキーに機能を割り当てて抜けれるときに `<Tab>` で閉じ括弧や閉じクォートを抜けるようにするには：
```vim
inoremap <silent><expr><Tab>  tozzy#is_leavable() ? tozzy#leave() : "\<Tab>"
```

- - -
### 変数
#### `g:tozzy_def`
初期値： ``{'*': ['"', "'", '`', '( )', '[ ]', '{ }'], 'vim': [' `']}``

初期値は変更されることがあるので正確には `plugin/tozzy.vim` でご確認ください。


`g:tozzy_def` は辞書で、キーには適用したいバッファのパターン(ファイルタイプまたは拡張子）、値には文字列のリストを設定します。
このリストにはクォートかペア（括弧など開き文字と閉じ文字を持つもの）を表す文字列を設定します。

ペアは半角スペースを挟んで定義します。`"[ ]"` なら開始文字は `[`、終了文字は `]` です。半角スペースで分かれていない文字はクォートと見なされます。クォートはペアとルールや挙動が異なります。

マルチバイト文字や複数の文字からなるペアも指定できます。(ただしIMからの入力にはまだ不具合があります)

設定例：
```vim
let g:tozzy_def = {}
let g:tozzy_def["*"] = ['"', "'", '( )', '[ ]', '{ }', '「 」', '（ ）', '【 】']
let g:tozzy_def["html"] = ["<!--  -->", "/**  */"]
```


##### `g:tozzy_def` の辞書キー（バッファパターン）について
```vim
let g:tozzy_def = {}
let g:tozzy_def["*"] = ['"', "'", '( )', '[ ]', '{ }']
let g:tozzy_def["html|*react|.jsx"] = ['< >']
let g:tozzy_def["html"] = ["<!--  -->", "/**  */"]
```

`*` だけの辞書キーはグローバルな設定です。個別な設定と被っていた場合、個別な設定の方が優先されます。

個別な設定にはファイルタイプ（オプション 'filetype' で設定されるもの）か、拡張子が指定できます。`|` 区切りで複数指定できます。

ファイルタイプ名では `*` をワイルドカードとして使えます。

`.` から始まる名前はファイルタイプではなくバッファ名の拡張子でマッチさせます（拡張子でマッチさせるときにはワイルドカードは使えません）。`.jsx` なら拡張子が `jsx` であるバッファにマッチします。

したがって、`g:tozzy_def["html|*react|.jsx"]` は3種類の定義のどれかにマッチすればその設定が適用されます。すなわち、ファイルタイプが `html` のもの、 `javascriptreact` `typescriptreact` のように末尾に "react" を含むファイルタイプ、または拡張子が `.jsx` であるバッファにマッチします。

加えて、ファイルタイプ `html` は、その後に個別で `g:tozzy_def["html"]` も定義されているので、その設定も適用されます。

##### `g:tozzy_def` の値のリスト内の文字列の表記ルール（クォート・ペアの表記ルール）
ペアは半角スペースを挟んで定義します。'( )' なら開始文字は '('、終了文字は ')' です。半角スペースで分かれていない文字はクォートと見なされます。
###### 先頭が空白で始まる定義は否定と見なされます。
```vim
let g:tozzy_def = {}
let g:tozzy_def["*"] = ['"', "'", '`', '( )', '[ ]', '{ }']
let g:tozzy_def["vim"] = [' `']
```

全体ルールでは `` ` `` が定義されていますが、ファイルタイプ `vim` では `` ` `` をクォートとして扱いたくないため、半角スペースから始まる定義で打ち消しています。

###### 末尾が空白で終わる定義は即時展開されます。
```vim
let g:tozzy_def["*"] = ['{ } ']
```

通常、`{` の後に任意の一文字を入力しないと `}` は補完されませんが、`"{ } "` のように設定していると、`{` を入力すると同時に `}` が補完されます。ほかのプラグインと同じような挙動になります。

##### 特殊文字列 `&&[ ]` はコレクションです。
`&&[abc]` は `a` `b` `c` の文字いずれか、`&&[a-zA-Z]` は任意のアルファベット一文字に対応します。これはペアの定義の終了文字側には使えません。

Vimの正規表現の collection を使っているのでそれと同じように、文字 `]` `^` `-` `\` は特別な意味を持ちます。これらをコレクション内で使うにはバックスラッシュでエスケープします。

```vim
let g:tozzy_def[".ejs"] = ['<% %>', '<%&&[=\-#%] %> ']
```
これはバッファ名拡張子 `.ejs` のバッファにおいて、'<% %>' と、'<%= %>' '<%- %>' '<%# %>' '<%% %>' に展開されます。

Pythonの `f"..."` `r"..."` `b"..."` `u"..."` など頭にアルファベットが付いた文字列に使う場合の例：
```vim
let g:tozzy_def["python"] = ['&&[frbu]&&["'']']   " クォートとして
```

Rubyの %記法に使う場合の例：
```vim
let g:tozzy_def["ruby"] = ['%&&[qQwWiIxs]&&[!`]', '%&&[qQwWiIxs]{ }', '%&&[qQwWiIxs]( )']
```

##### 即時展開とコレクションの組み合わせ
例えば、Vim scriptにおけるキーマップのときだけ角括弧を使いたいとします（`<C-f>` のような）。

しかし、そのまま `"< >"` のように定義すると `<` をほかの用途で、例えば `<=` のような不等号比較演算子を使うときに邪魔になります。

その場合、例えば以下のように定義します。
```vim
let g:tozzy_def["vim"] = ['<&&[a-zA-Z] > ']
```
キーマップに使うときの角括弧は `<` の隣が半角英数なので、コレクション `&&[a-zA-Z]` で対象にします。定義の末に空白を含めることで即時展開するようにして、`<` の後に半角英数が入力されたときに閉じ括弧を補完します。

- - -
#### `g:tozzy_inhibition_pat`: 閉じ文字補完の抑制
規定値： `{'vim': '^\s*".\%#\|" \%#$'}`

`g:tozzy_def` は辞書で、キーには `g:tozzy_def` のようにファイルタイプ名やファイル拡張子を指定し、値にカーソルパターン `\%#` を含むパターン文字列を指定します。

このパターンにマッチしたときには閉じ文字の自動挿入を抑制します。

例えば、ファイルタイプ `vim` では、`"` はクォートのほかにコメントの開始文字としても使用されるのでそのように使用したいときにクォートとして展開されたくないとします。

まず、行の先頭に `"` を入れるときには必ずコメント文字となるので、そのパターンは `^\s*".\%#` となります。（カーソル`\%#` の前に任意の一文字 `.` を入れているのは実際に当プラグインで展開されるタイミングはクォートの後に任意の一文字を入力するときだからです）

次に行末にコメントを入れたいとき、通常、コメントの後にはコメントアウトと区別するためにスペースを一文字挟む慣習があるので、`"` の後にスペースを入力したパターン `" \%#$` となります。

これらを `\|` で繋いで以下のように設定すれば、コメントとして `"` を入力するときの閉じ文字補完を抑制できます。
```
let g:tozzy_inhibition_pat = {}
let g:tozzy_inhibition_pat["vim"] = '^\s*".\%#\|" \%#$'
```

## クォートとペアの違い
### クォート
シングルクォート `"x"` には対応しているが、二重クォートや三重クォート `""x""` `"""x"""` にはデフォルトでは対応していません。。これは誤爆しやすいからです。

#### 長さ2以上の文字列からなるクォートの定義では最後の文字が閉じ文字として使われます
```vim
let g:tozzy_def["ruby"] = ["%Q!"]
```
この場合 `!` が閉じ文字として使われます。`%Q!` の後に `foobar` を入力すると、展開結果は `%Q!foobar!`

```vim
let g:tozzy_def["python"] = ['f&&["'']']
```
この場合コレクション文字 `"'` のうち、実際に開始文字として使われた方が閉じ文字として使われます。`f"` で入力があれば閉じ文字は `"`

#### クォート対象条件
1.左隣が半角英数やアンダースコアやバックスラッシュの場合、対象にならない。(英文の使用やエスケープされたクォートへの対応のため)
  ```
  start: |   【input: `It'`】→ It'| 【input: `s crazy.`】→ It's crazy.
  start: "|" 【input: `\`  】→ "\|" 【input: `"xx`】     → "\"xx|"
  ```
2.右隣にすでにクォート閉じ文字がある場合は対象にならない。（閉じクォートが既に入力されている状態で開きクォートだけを入力するケースに対応するため）
  ```
  start: |" 【input: `"`】 → "|" 【input: `x`】 → "x|"
  ```
3.左隣にすでにクォート開始文字がある場合は対象にならない。（開始文字が入力されている状態で閉じ文字だけを入力するケースに対応するため）<br>ただし、文字が2つ重なるクォートの定義がある場合はこの限りではない。
  ```
  # '""' という定義がない場合（'"' という定義ならあるが二重クォートの定義はない）
  start | 【input: `""`】 →  ""| 【input: `x`】 →  ""x|
  # '""' という定義があるのなら
  start | 【input: `""`】 →  ""| 【input: `x`】 →  ""x|""
  ```

### ペア
クォートと違って二重括弧や三重括弧に対応しています `((x))` `(((x)))`。同種の開き文字を連続で入力しているときは補完されず、閉じ文字か、任意の文字の入力によって初めて補完されます。閉じ文字が入力された場合はすべての閉じ文字を補完してペアの外に出ます。
```
| 【input: `{{`】 → {{| 【input: `}`】 →  {{}}|
| 【input: `{{`】 → {{| 【input: `x`】 →  {{x|}}
```

開き文字入力の後で改行すると、閉じ文字を次の行に作ります。
```
| 【input: `{`】 →  {|  【input: `<CR>`】→

{
  |
}
```

#### ペア対象条件
1.開き文字の右隣が `[0-9a-zA-Z_]` の場合、対象にならない。（数字や変数名を後から括弧でくくるケースに対応するため）

2.開き文字の最初の文字が `[0-9a-zA-Z_]` の場合、その左隣は `[0-9a-zA-Z_]` であってはいけない。（始まりの区切りが曖昧になるため）


## 特別な挙動
以下の場合、補完はキャンセルされ、通常にそのような文字を入力したときと同じ結果になります。
- 開き文字を入力した後即座に閉じ文字を入力した場合
  (例：`|` 【input: `[`】→ `[|` 【input: `]`】→ `[]|`)
- 開き文字を入力して一文字だけ何かを入力して閉じ文字を入力した場合
  (例：`|` 【input: `[`】→ `[|` 【input: `x`】→ `[x|]` 【input: `]`】→ `[x]|`)


## Author
seroqn

## License
MIT
