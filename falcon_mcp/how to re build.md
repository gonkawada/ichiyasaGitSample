docker-composeで対象のコンテナだけを再ビルドし、volumeマウントを廃止して今後は修正済みイメージで運用する場合の手順は以下の通りです。[1][2][3]

### 1. docker-compose.ymlのvolumes設定を削除
- 修正対象のサービス（たとえば`app`など）から`volumes:`設定を削除してください。
- これでホストとコンテナ内のソースコード同期（マウント）は行われなくなります。

### 2. ソースをDockerイメージにコピー
- Dockerfile内で`COPY`命令などを使い、修正済みソースコードをイメージ内に取り込みます。
  ```
  COPY ./src /app/src
  ```

### 3. イメージの再ビルド
- 対象サービス（ここでは`app`と仮定）のみ、下記コマンドでビルドします。
  ```
  docker-compose build --no-cache app
  ```
  `--no-cache`を付けることでキャッシュを使わずイメージを再作成できます。[3]

### 4. コンテナの再作成・再起動
- サービスを再作成して起動し直します。対象サービスのみなら、
  ```
  docker-compose up -d --force-recreate --no-deps app
  ```
  これで他のサービスには影響しません。

### 5. 動作確認と不要ファイル/volumeの掃除
- 動作を確認し、不要なvolumeがあれば下記で削除します。
  ```
  docker volume ls
  docker volume rm <不要なvolume名>
  ```

### まとめ
この流れにより、ホストとの同期をやめ、Dockerイメージで管理された状態に移行できます。今後はソース修正ごとにイメージをビルドし直せばよばよいため、volumeマウントは不要です。[8][6][10]

[1](https://zenn.dev/yasuhiron/articles/rebuild-a-specific-container-in-docker-compose)
[2](https://watashi.xyz/docker-compose-up-options/)
[3](https://www.it-educationblog.com/entry/2024/01/12/130957)
[4](https://qiita.com/Dai_Kentaro/items/de26054e8cf1e019a667)
[5](https://docs.docker.jp/compose/compose-file/compose-file-v3.html)
[6](https://future-architect.github.io/articles/20240726a/)
[7](https://zenn.dev/yuji_momotani/articles/d67798bd2e68a6)
[8](https://www.northtorch.co.jp/archives/1792)
[9](https://deep.tacoskingdom.com/blog/223)
[10](https://scrawledtechblog.com/docker-docker-compose-volumes/)
[11](https://qiita.com/suin/items/e53eee56da23d476addc)
[12](https://www.reddit.com/r/docker/comments/sxz70d/what_should_i_run_after_making_changes_to/)
[13](https://engineering.nifty.co.jp/blog/24155)
[14](https://ysko909.github.io/posts/delete-volume-if-can-not-update-container/)
[15](https://labor.ewigleere.net/2023/07/21/excludes-part-of-volumes-in-mount-volumes-on-docker-compose/)
[16](https://zenn.dev/yuki_tu/scraps/d0acfa5c552ca1)
[17](https://www.slideshare.net/slideshow/docker-compose-guidebook/134824655)
[18](https://qiita.com/aminosan000/items/6f1d1b1e730e01907fbc)
[19](https://www.reddit.com/r/docker/comments/12ha2wf/docker_compose_setting_readonly_bind_mount/)
[20](https://qiita.com/colorrabbit/items/8afd49966c4e9dc393f6)
