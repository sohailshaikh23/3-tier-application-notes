########################## Stage 1: build stage as backend-builder #####################
FROM node:14 AS backend-builder

LABEL maintainer="myemail@gmail.com"

# setting up the working directory as /app
WORKDIR /app

# copy the application source code 
COPY . .

# Installing all the dependencies
RUN npm install


############################### Stage 2: Final stage ######################################
FROM node:21-slim

#Setting up the current working directory
WORKDIR /app

#Copy only the require dependency file from the backend-builder stage to the current stage
COPY --from=backend-builder /app .

# Exposing the port 8080 for the application
EXPOSE 8080

# provide the Entrypoint command to run the application
ENTRYPOINT [ "node" ]

# provide the default command to run the application
CMD ["index.js"]








