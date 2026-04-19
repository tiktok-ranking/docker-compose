# NOTE: 第2章・第3章の実装が完了するまでの暫定ダミーサーバー
FROM nginx:alpine

# ダミーのHTMLファイルをコピー
COPY index.html /usr/share/nginx/html/index.html

# 空白の数に依存しないよう正規表現でポート 3000 へ書き換え
RUN sed -i -r 's/listen\s+80;/listen 3000;/g' /etc/nginx/conf.d/default.conf

EXPOSE 3000