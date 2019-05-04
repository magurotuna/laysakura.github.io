## blogの場所

https://laysakura.github.io

## 記事作成の流れ

### 既に手元の端末で `hexo` コマンドが使えるようになってるとき

1. 最新の `ready` ブランチから `article/xxx` ブランチを切り出す。
2. `hexo new '記事のタイトル'` で記事のひな形を生成し、エディタで記事を書く。
3. `hexo server` で localhost:4000 を待ち受けるので、ブラウザで見た目確認。
4. `git add source/_posts/ ; git commit ; git push origin article/xxx`
5. https://github.com/laysakura/laysakura.github.io で `article/xxx` から `ready` に向けたPRを作成し、マージ。
6. `hexo deploy --generate` すると、よしなに記事が公開される。

### `hexo` のインストールから始めるとき

`hexo init` から始めないと、 `hexo` の依存パッケージ感の相性が悪く `hexo new` などに失敗する。

```bash
git checkout ready

rm package*.json
rm -rf node_modules/

hexo init ../tmp

cp ../tmp/package*.json .
cp -r ../tmp/node_modules .

hexo server  # 動作確認
```

## テーマ変更の流れ

git subtree で `themes/apollo` を作っている。

```bash
emacs  # themes/apollo/ 以下のファイルに変更を加える
git add themes/apollo/
git commit -m 'foobar'

cd ..

# このレポジトリへのpush
git push origin ready

# laysakura/hexo-theme-apollo レポジトリに feature/foobar ブランチとしてpush
git subtree push --prefix=themes/apollo git@github.com:laysakura/hexo-theme-apollo.git feature/foobar
```

https://github.com/laysakura/hexo-theme-apollo でPR & mergeしておく。

## トラブルシュート

- `themes/xxx/` でCSSを更新したのに反映されない
    - `hexo clean && hexo deploy --generate`
- `hexo deploy` で `ERROR Deployer not found: git` と言われて失敗する
    - `npm install hexo-deployer-git --save`
