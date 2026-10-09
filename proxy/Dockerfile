# Proxy image inherited from the previous team.

FROM nginx:1.27-alpine-slim

COPY nginx.conf /etc/nginx/nginx.conf

RUN chown -R nginx:nginx /var/cache/nginx /var/log/nginx

USER nginx

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]