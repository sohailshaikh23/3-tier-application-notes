FROM node:14
LABEL "myemail@gmail.com"
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ENTRYPOINT [ "node" ]
CMD ["index.js"]
