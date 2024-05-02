# Stage 1: Build stage named as frontend-builder
FROM node:14 AS frontend-builder

# information about maintainer
LABEL maintainer="my@gmail.com"

# Setting up the current working directory as /app
WORKDIR /app

# Copy the rest of the application code
COPY . .

# installing the dependencies
RUN npm install


######################################### Stage 2 : Final stage ######################################
FROM node:14-slim

# Setting up the current working directory as /app
WORKDIR /app

# copy only the require dependencies from the frontend-builder stage to the Current stage
COPY --from=frontend-builder /app .

# Exposing the port 3000 for the application
EXPOSE 3000

# Provide the default command to run the application
ENTRYPOINT [ "npm"]

CMD [ "start" ]
