# UMass 2023 - js\_on

## 💙 Description

> Why call it JSON if I can't put my own JS there? Introducing JS-ON! Now you can share your JS with your friends to enjoy next level development!
>
> Bot: [http://bot-json.web.ctf.umasscybersec.org:9000](http://bot-json.web.ctf.umasscybersec.org:9000)
>
> File : [app.js](https://umass-ctf-challenges.s3.amazonaws.com/web/app.js)
>
> Main Site : [http://app-json.web.ctf.umasscybersec.org:3000](http://app-json.web.ctf.umasscybersec.org:3000)

## The Challenge

What we first run into by looking at this challenge is a website that allows you to run code with compiled `JS-ON` for an user.

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>The main website</p></figcaption></figure>

There is also a 'bot' website that seems to run the user's generated code as admin.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption><p>The Bot website</p></figcaption></figure>

Based on the code provided, the application is a web server that allows users to write JavaScript code and execute it on the server-side using a technology called JS-ON.

The server creates a unique identifier as a cookie for each user that connects to it, and saves the user's code and JS-ON payload in Redis. Users can retrieve their code and payload by specifying their unique identifier in the URL path.

There is also an admin feature that allows a user with a specific cookie to access a pre-defined library of JavaScript code with an embedded flag. However, this feature is protected by a cookie, which is checked against a predefined value that we can't access.
