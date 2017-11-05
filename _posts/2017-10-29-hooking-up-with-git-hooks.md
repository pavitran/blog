---
layout: post
comments: true
title: "Hooking up with Git Hooks"
quote: Don't wanna get hooked? be the fish that swims in solitude
image: /blog/media/2017-10-29-hooking-up-with-git-hooks/git.png
video: false
---
Even though the title is a bit weird, you might have already guessed what this article is about. Yeah, "Git" Sometimes no matter how much you explore a tool or a software, there are still surprises in its pocket. One such tool is Git. Yup, the nifty Version Control System(VSC) which is most popular and most used among software developer(the smart ones), It might have saved lots of butts from getting busted, at least it did save mine a lot. Don't know how many lives those `git reset`,`git revert` and `git checkout` has saved.
Use Git Properly and it makes your life as a software developer so much easier, and If you are not using it already be prepared for the horrors. But I’m not here to praise Git, am I? I'm here to talk about a feature that was there forever but I've come across it in the recent days.
I'm talking about `Git Hooks`, What are they? How could they of be use to you? How I've used them? Let's find out..

`Git Hooks` are just Bash Script files that you could execute whenever a certain event occurs. Well, What kind of events exactly?🤔
When using `Git` there are a lot of events.    

  - we run `git add .` to stage the files for the commit, that's an event  
  - we store the changes to the local repo with `git commit -m "first commit"`, this is an event  
  - we push the changes to the remote repo with `git push`, that's an event as well

For this article, let's concentrate  on the last two. Now let's see some of the cases where `Git Hooks` might be useful..
### Use cases  
1. **changing credentials when committing code**  
 This happens to me often, I've been working on website locally firing up a local database server, and have my app connected to it. My development is done, and again when I'm commiting the code I've to change the database link, credentials manually. Not much of a hassel, but could be automated.
2. **prettifying code before committing**  
Whenever I'm writing code my top most prioroty is getting it to work, So I don't care about those beautiful indentations and perfectly alligned curly braces. But since I'm sharing code with others, before comitting it I have to go on and adjust all the code to make it look pretty. this is somewhat tidious job, so should be automated.(or rather learn to write nicely intdented code stupid😆)
3. **custom notification system for push**  
Github has many kinds of notifications, so whenever one of your collaborator has pushed some new changes you get a notification. But wait what if you are not using Github and have hosted your own remote repo, or what if you are using Github but still want to be notified of changes by some other means. We can use `Git Hooks` for that as well.

Well mate we've talked all about what  they can do, but there's still one thing that's bugging me. Why haven't I seen them even though I've been working with Git for a while?  
Good question! Well they have been there forever hiding under plain sight. Now let's get into some technical stuff, let's see how we can write `Git Hooks` with the above three Usecases in mind.  
When you do a `git init` or a `git pull`, Git creates a directory called `.git` in the root directory. This is where all the data regarding our repo are stored. If you can't see the directory, it's probobly been hidden(by default). Unhide the directory according to your operating systeam and get into it, you'll see that it has the following file structure  

```
.git
└─── hooks
└─── info
└─── logs
└─── objects
└─── refs
│   COMMIT_EDITMSG    
│   config    
│   description
│   HEAD
│   index

```  
Here `hooks` directory is all we care about, but let's briefly go through what all these files and directories are about. `index` file contains all the details about the files the directory, `HEAD` contains the name of the head we are at, `description` contains the description of our repo like the name, `config` obviously contains the configuration of out repo, `COMMIT_EDITMSG` contains the commit message of our last commit.  
Now Let's come to the directories, `refs` directory contains the raw SHA1 values, `objects` is the directory where all the Blobs with the info of commits are stored, `logs` are again logs and `info` is just info.
At Last we come to the star of our show the `hooks`, open up the directory and you'd see a bunch of file with an extension `.sample`. The names of files map to kind of event that triggers the hook, for example `pre-applypatch.sample` would be executed before a new patch is being applied, whereas `post-checkout` executed after we have run `git checkout`.  
Now let's take the use cases from above and let's write three different `Git Hooks` that help us in three different ways..    

1. **Different credentials for dev and production**  
  I usually have different credentials file for dev and production, let's call them `credentials.json` and `credentials-dev.json`. Now we usually have to switch them before commiting the code, let's automate it with hooks..  
 - In the current `hooks` directory you'd have a file with the name `pre-commit.sample`, remove the `.sample` extension, save and open up the file in your favourite text editor. You should see some sample bash script, let's ignore it for now and remove all the content's of the file(except the `#!/bin/sh`). Now let's assume you are working with a node project and in your project directory have a `index.js` file, this is the file in which we require the credentials. Let's switch the occurances of `credentials-dev.json` with `credentials.json` using sed..  
     ```
        sed -i "s/credentials-dev.json/credentials.json/g" ./src/index.js  
        git add ./src/index.js  
      ```
  This would replace all the occurance of our dev credentials with production credentials, and the next line `git add ./src/index.js` to add the chaged file back to the staging, you could do `git add .` if you are updating multiple files in the hooks. Save it, try commiting the code and you should see the changes in the file.  
 - We just switched the credentials to production, but wait again for development you have to switch them back manually, seems inefficient🤔. Let's improvise it. Create a new hook that changes back the production credentials to dev, in same `hooks` directory create a new file `post-commit`(without any extension). open the file in editor and add `#!/bin/sh` on top, again just one line which does the exact opposite.  
  ```
  sed -i "s/credentials.json/credentials-dev.json/g" ./src/index.js
  ```
 - That's all, try commiting the code and you should see no changes in the `index.js`. Why? Because even though we change it before the commit with `pre-commit` hook, we changed it back with `post-commit` hook. Do a quick `git diff` and you should see that the last commit really did change to production credentials.  

 2. **Prettifying code before committing**  
This is another common one, I sometimes start with the prittiest code possible and as I progress it turns into something else. Let's use a `pre-commit` hook for this
 - Since we are working with node, there's an awesome node module [prettier](https://github.com/prettier/prettier#option-3-bash-script), that makes all your javascript prettier. install it with `npm install --save-dev --save-exact prettier`, now let's go to our `pre-commit` hook file and add the code that iterates over all the js files and makes them "pretty".  
```
jsfiles=$(git diff --cached --name-only --diff-filter=ACM | grep '\.jsx\?$' | tr '\n' ' ')
[ -z "$jsfiles" ] && exit 0
# Prettify all staged .js files
echo "$jsfiles" | xargs ./node_modules/.bin/prettier --write
# Add back the modified/prettified files to staging
echo "$jsfiles" | xargs git add
exit 0
```
We aren't doing anything fancy here, just copy pasting the code from docs.
  - Above I showed you a how you can do it for javascript, but you can achive the same using respective modules in your programming language.

3. **custom notification system for push**  
The last usecase we'll be talking about is custom notifications on `git push`, as I explained earlier this could be used for send notifications when git repo is in local network or if not using git based platforms like [Github](https://github.com/). Let's do this..
  - Firebase is another product I use in many of my projects. It's like a swiss army knife for app developers, It's got it all in it's pockets.. whether it's realtime database, storage, push notifications, analytics or cloud functions.(what else could you ask for!)
  - For this we'll need two of firebase products `realtime database` and `firebase functions`, `firebase function` Implementation would be out of scope of this blog post. Neither will I tell you how to setup the firebase project, The only firebase part we'll be talking about is how you can add data to `firebase realtime database` using REST API from Git Hook.  
  Hey Man! This is not fair, now I have to learn about a whole new platform just to see how a `git push hook` work?  
  Don't worry! since we'll be using `curl` to add data to database using REST API, you can use the same knowledge with any other platform.
  - Create a new file `post-push`(again with no extension), and save it in the hooks directory. Add the `#!/bin/sh` header and save it.
  - You all know what [curl](https://curl.haxx.se/) is right ? Now we'll send a json object with the name of the author and time of push to the `firebase database` REST API endpoint.  
    ```
    now=$(date +"%T-%d/%m/%Y")<br />
    curl -X PUT -d '{ "name": "Pavitran", "time": "'"$now"'" }' \
      'https://git-hooks-15794.firebaseio.com/commits.json'
    ```  
    In the above code, we get the time, date and save in the `$now` variable. Then using curl we Put json data with a `name` and `time` properly, to the endpoint `https://git-hooks-15794.firebaseio.com/commits.json`. This will add the data to the `firebase database`.
  - Hey But wait, you said we were going to send a notification and all you did was add some data to the database?  
  Well I did, but Implementating the remaining part would be out of scope. But if you still want to, All you have to do is listen for changes in database from firebase function and send a notification through email. If you are interested you can checkout this example [email-users](https://github.com/firebase/functions-samples/tree/master/quickstarts/email-users), where you can learn how to send an email using firebase function.

  That was `Git Hooks` in a nutshell, Do let me know your views on this piece. Also let me know if you got any improvisations or suggestion in the comments below. That's it for now, signing out!
