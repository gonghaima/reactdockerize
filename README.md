Dockerizing a React application
Late last year I was working with our Semantic Scholar team to migrate the online stack from a bare-metal system to a system running containers in Docker. We set out with the main goal of running development builds using Docker, with the same images as we’ll use in production. This fixes a potential problem in development builds: a mismatched node version causing differing behavior between developers. It also gives us very high confidence in the developed system, since we’ll be matching our production system so closely while developing.

This tutorial walks through Dockerizing for production deploys, but doesn’t go into the details of Dockerizing the full development stack. I’ll leave that for a future post.

Before going into the details of what we did, there’s one important side note: our primary development environment is OS X. You might see slightly different performance running Docker on Linux, since it runs fully-native.

All of these examples all assume you have access to npm and docker on your commandline. For OS X, Docker for Mac is easy-to-install and (generally) works great.

Initial (Naive) Approach
Let’s review our requirements: We want all developers to run the same version of node, and we want images to contain all the files needed for serving our application. We also, obviously, need a React application! For this post, I’ve created a new application using create-react-app:

$ npm install -g create-react-app
$ create-react-app dockerize-me
(lots of output over lots of minutes)
I’m also going to go ahead and eject out of this tooling immediately, because I’ll be updating some of the build files to work better with Docker:

$ cd dockerize-me
$ npm run eject
There’s one final thing we need to do: Work around a bug in npm where it tries to install optional dependencies. Add the following to your package.json file, before devDependencies:

"optionalDependencies": {
  "fsevents": "*"
},
We also want to generate a shrinkwrap file, since this ensures all of our developers use the same versions of dependency libraries. Note that create-react-app will generate a non-shrinkwrappable dependency tree, so we clean it up first. We also use a --dev shrinkwrap, since we’ll be building in our Docker image and need the development dependencies to do so:

$ npm prune
$ npm dedupe
$ npm install
$ npm shrinkwrap --dev
Now that we have an application, we can start the Dockerizing! The first requirement we listed — pinning our node version — is super-easy to meet, since node has an official Docker image registered on Docker Hub. We create a file called Dockerfile in our project directory, and add the image we want to inherit from, and override the log level setting:

# In your Dockerfile.
FROM node:7.8.0
# The base node image sets a very verbose log level.
ENV NPM_CONFIG_LOGLEVEL warn
The second thing we need is to copy our project files into the image. When you run docker build, it automatically copies the full working directory into your workspace, so all you need to do is copy it into the image:

# In your Dockerfile.
COPY . .
You can then build it with a RUN command:

# In your Dockerfile.
RUN npm run build --production
Finally, we need to install an HTTP server for running the application. I use serve, the server that create-react-app recommends:

# In your Dockerfile.
RUN npm install -g serve
# Run serve when the image is run.
CMD serve -s build
# Let Docker know about the port that serve runs on.
EXPOSE 5000
The final Dockerfile, with some added comments, should look something like this:


Now you can build your image! From the shell, in your project directory, run docker build -t react-docker .:

$ docker build -t react-docker .
Sending build context to Docker daemon 88.76 MB
Step 1/6 : FROM node:7.8.0
*snip*
Successfully built 0123abcd
Hrm. That took a long time. We’ll work on fixing that after we run our image! Verify that it’s working by running docker run -it --rm -p 5000:5000 --name react-demo react-docker.

You should see serve tell you that it’s serving traffic on localhost:5000 (it’s actually serving in the container, but we’ve mapped it to our localhost with the -p flag). You should be able to load it in your browser.

So we’re done, right? Not quite! We made a couple of mistakes with how we set up the build. The biggest mistake: We copied in our local node_modules directory instead of building it fresh inside the image! This means that we failed one of our primary goals — making the server build with the same version of npm for everyone. It also makes the build substantially slower, due to copying the 88 (!) MB of data into the Docker context with each build. You can see that if you re-run the build command from earlier:

$ time docker build -t react-docker .
Sending build context to Docker daemon 88.76 MB
*snip*
        5.34 real         0.54 user         0.93 sys
Even with a fully-cached build, this still takes over 5 seconds on my laptop!

The other problem is with the ordering of our commands in our build. Docker uses a cache when building images, and we’ve structured our Dockerfile in such a way that it can’t effectively cache the result of some of our operations. You can see this in action if you change a source file (like src/App.js and re-run the build command:

$ time docker build -t react-docker .
Sending build context to Docker daemon 88.76 MB
*snip lots of output*
       32.71 real         0.58 user         0.78 sys
You’ll notice that this rebuilds the application, as expected, but is also reinstalls serve, which is silly.

Fixing Our Dependencies
Fixing our first issue, dependencies copied from the build host, is quite simple: All we need to do is run npm install in our Dockerfile. However, if we’re doing that, there’s no reason to copy the full node_modules directory over at all. That’s where a .dockerignore file comes in. This lets you filter which files the Docker CLI sends to the Docker daemon, which is great for our efficiency!

For starters, create a file called .dockerignore in your project directory with the following contents:


That will keep us from sending all of our local dependencies to the daemon, which speeds up the startup a ton! You can see this by running the build command again:

$ time docker build -t react-docker .
Sending build context to Docker daemon 261.6 kB
*snip*
npm WARN Local package.json exists, but node_modules missing, did you mean to install?
*snip*
The command '/bin/sh -c npm run build' returned a non-zero code: 1
        3.01 real         0.02 user         0.02 sys
Whoops! You’ll also see that we can’t run npm build, either, for obvious reasons. To fix this, we need an npm install. Now, could just RUN this below our existing COPY command, but we’re going to do something smarter instead.

The problem here is with our naive COPY of everything in our working directory. This is great for the build step, but it’s far more than what we need for the install step. Furthermore, Docker is smart enough to cache the result of each command in a Dockerfile if the contents haven’t changed between runs. This means that we can skip the npm install run on most builds if we are careful. Since all that npm install requires is package.json and npm-shrinkwrap.json, we only need to copy those files over:

# Put these immediately above the 'COPY . .' in your Dockerfile.
COPY package.json package.json
COPY npm-shrinkwrap.json npm-shrinkwrap.json
RUN npm install
Now you can run your build again. Note that you’ll see a warning about the missing fsevents package; this is fine to ignore:

$ time docker build -t react-docker .
*snip*
       67.96 real         0.02 user         0.02 sys
Whew, that took a long time! However, try editing App.js and building again:

$ time docker build -t react-docker .
*snip*
       20.58 real         0.02 user         0.02 sys
Much faster! But still quite slow — we wasted a bunch of time installing serve again. Fortunately, we can fix this! The best-practice with a Dockerfile is to place common / unchanging commands at the very beginning, and the most-frequently changing items at the end. With that in mind, let’s move the basic command configuration to the very top. You should end up with a file something like this:


Now our incremental builds should be as fast as we can make them:

# Initial build.
$ time docker build -t react-docker .
*snip*
       69.43 real         0.02 user         0.01 sys
# Edit App.js and rebuild . . .
$ time docker build -t react-docker .
*snip*
        9.61 real         0.02 user         0.01 sys
Much better! Watching the logs, you should also have seen that all of the time was taken up in building the optimized production build — exactly what we were hoping.

In addition to speeding up the build, this also reduces the space each revision takes up on your Docker server and your Docker registry. Docker stores each of the commands separately, and each time it says “Using cache” in the output, you’re saving storage space. This doesn’t matter for smaller layers (like your build output), but it can matter a lot for those 80+MB of node packages!

Conclusion
Hopefully this is a helpful document for someone trying to Dockerize an existing React application, even one that doesn’t use create-react-app. Our build on Semantic Scholar is heavily customized, and wasn’t ever set up with create-react-app, so only some of our work directly translated to this post. Please feel free to point out any errors I made, or any issues I might not have noticed with the suggested setup!

On Semantic Scholar, we also went one step further than what I showed here and ran our development servers within a Docker container. However, we found that the performance of the actual development build itself was abysmal, so that stayed within the local OS. What we did to make this work is interesting, but the subject of another post. :)