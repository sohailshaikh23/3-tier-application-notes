FROM node:14
WORKDIR /3-tier_app
COPY Dockerfile .
COPY package*.json .
RUN npm install
COPY . .
EXPOSE 3000
ENTRYPOINT [ "npm"]
CMD [ "start" ]
