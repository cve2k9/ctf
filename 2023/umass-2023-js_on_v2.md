# UMass 2023 - js\_on\_v2

## 💙 Description

> A little birdie told me that there are some unintended solutions to the challenge I worked for so long on! However I'm on a time crunch and can't put a patch myself so I let ChatGPT write me one. If your unintended (or intended) solution from the unpatched version works, enjoy a free 500 pts!
>
> Bot: [http://bot-json.web.ctf.umasscybersec.org:9000/](http://bot-json.web.ctf.umasscybersec.org:9000/)
>
> File: [https://umass-ctf-challenges.s3.amazonaws.com/web/v2/app.js](https://umass-ctf-challenges.s3.amazonaws.com/web/v2/app.js)
>
> Main Site: [http://app-json2.web.ctf.umasscybersec.org:3000/](http://app-json2.web.ctf.umasscybersec.org:3000/)

## :purple\_heart: Note

This challenge is effectively the same as [JS-ON](js\_on.md). Go read that first.

## The Challenge

Through some unknown AI-powered means the challenge is now supposedly more secure, ruling out previously unintended solutions.

## The Solution

Starting off with a diff of the frontend code, we can see that not much has changed. All `var_` JS-ON entries now have `<` / `>` stripped from their content, but apart from that everything is the same, which also means our `Function()` overwrite still works.

```diff
diff --git a/./a.js b/./b.js
index fba5c53..8da1bd8 100644
--- a/./a.js
+++ b/./b.js
@@ -31,10 +31,13 @@
                             let keywords = key.split('_');
                             let type = keywords[0];
                             let name =  keywords[1];
                             switch(type){
                                 case 'var':
+                                    if(typeof JSON[key] === 'string'){
+                                        JSON[key] = JSON[key].replaceAll('<','').replaceAll('>','');
+                                    }
                                     cur[name] = JSon[key];
                                     break;
                                 case 'func':
                                     if(!(cur[name] in cur)){
                                         cur[name] = Function(JSon[key])
```

In the backend, there is now a blacklist of certain words that may not be used within the keys of JS-ON environments. We don't exactly know what words are banned though.

<pre class="language-diff"><code class="lang-diff">diff --git a/./app.js b/./app.js.1
index 377f1dd..61f5883 100644
--- a/./app.js
+++ b/./app.js.1
@@ -4,10 +4,13 @@ const cookiep = require("cookie-parser");
 const path = require('path');
 const { v4: uuidv4 } = require('uuid');
 const redis = require('redis')
 const port = process.env.SERVER_PORT;
 
+const BANNED = ['banned list on server! I don\'t want to spoil the chall!'];
+
+
 
 const client = redis.createClient({
   'url':process.env.REDIS_URL
 })
 
@@ -70,24 +73,47 @@ app.get('/code/:id',async (req,res)=>{
   res.json(JSON.parse(user))
 })
 
 app.use(express.json());
 
+function checkObjectForBannedKeys(obj) {
+  for (const key in obj) {
+    if (obj.hasOwnProperty(key)) {
+      let keywords = key.split('_');
+      if (BANNED.includes(keywords[1].toLowerCase())) {
+        return false;
+      }
+      if (typeof obj[key] === "object" &#x26;&#x26; !Array.isArray(obj[key])) {
+        if (!checkObjectForBannedKeys(obj[key])) {
+          return false;
+        }
+      }
+    }
+  }
+  return true;
+}
+
 app.post('/code',async (req,res)=>{
   let uid = req.cookies.user ? req.cookies.user : '';
   let user = await client.get(uid);
   if(!user){
     return res.json({'error':'Could not find js-on for that user!'});
   }
   try{
<strong>     req.body['js-on'] = JSON.parse(req.body['js-on'])
</strong>   }
   catch(e){
     return res.json({'error':'Failed to parse js-on!'})
   }
-  await client.set(req.cookies.user,JSON.stringify(req.body));
-  res.json({'success':'JS-ON loaded onto server!'});
+  console.log(checkObjectForBannedKeys(req.body['js-on']))
+  if(checkObjectForBannedKeys(req.body['js-on'])){
+    await client.set(req.cookies.user,JSON.stringify(req.body));
+    res.json({'success':'JS-ON loaded onto server!'});
+  }
+  else{
+    res.json({'error':'BANNED WORDS DETECTED!'});
+  }
 })
 
 app.listen(port, () => {
   console.log(`JS-ON available on ${port}`);

</code></pre>

Messing around with re-submissions of our payload, it turns out that one of the banned words is "cookie" which blocks the `var_cookie` our current solution uses for phoning home. Changing this to something like `var_x` makes it work again. Yes really, that's all :)
