# Git is not fetching branch refs? Only master?

Apparently there's a config in git that manages which origins should it fetch. 
So if for some weird reason your `git fetch` is not fetching the remote refs you expect,
Check your git config
```
git config --get remote.origin.fetch
```

it's probably something like 
```
+refs/heads/master:refs/remotes/origin/master
```

instead of 
```
+refs/heads/*:refs/remotes/origin/*
````

You can change it to fetch all refs and heads with
```
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
```
