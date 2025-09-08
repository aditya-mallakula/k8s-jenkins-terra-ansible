FROM nginx:1.27-alpine
RUN mkdir -p /usr/share/nginx/html && printf 'hello from api\n' > /usr/share/nginx/html/index.html
EXPOSE 8080
CMD ["nginx","-g","daemon off;","-c","/etc/nginx/nginx.conf"]
