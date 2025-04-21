FROM node:18

WORKDIR /react

COPY ../react-expo/ .

RUN npm install -g expo-cli

RUN npm install axios

EXPOSE 8081

CMD ["expo", "start", "--web"]

