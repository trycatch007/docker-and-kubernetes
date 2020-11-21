# specify base image
FROM node:alpine

# set the default work directory
WORKDIR '/app'

# install some dependencies
COPY ./package.json ./
RUN npm install

# copy everything from local machine's <build contect> to the image
# after the npm install so it doesn't invalide its image cache
COPY ./ ./

# default command
CMD ["npm", "start"]
