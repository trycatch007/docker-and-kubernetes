# specify base image
FROM node:alpine

# set the default work directory
WORKDIR /usr/app

# install some dependencies
COPY ./package.json ./
RUN npm install

# copy everything from local machine's <build contect> to the image
COPY ./ ./

# default command
CMD ["npm", "start"]
