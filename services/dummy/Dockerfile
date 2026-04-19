# NOTE: 第2章のNext.js実装が完了するまでの暫定ダミーサーバー
FROM nginx:alpine

# ダミーのHTMLファイルをコピー
COPY index.html /usr/share/nginx/html/index.html

# 既存のリバースプロキシ設計（default.conf）に合わせてポート3000で待機
RUN sed -i 's/listen  80;/listen 3000;/g' /etc/nginx/conf.d/default.conf

EXPOSE 3000