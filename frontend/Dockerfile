# build
FROM node:18-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN if [ -f package-lock.json ]; then npm ci; else npm install; fi
COPY . .
RUN sed -i 's#http://localhost:8080/api#/api#g' src/app/services/tutorial.service.ts || true
RUN npm run build

# serve
FROM nginx:1.27-alpine
RUN rm -f /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/app.conf
COPY --from=build /app/dist/angular-15-crud/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
