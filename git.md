# Checking current git alias
```
git config --get-regexp ^alias
```

# Setting alias
```
git config --global alias.sa 'stash apply'
```

# Pure shorthand to save typing

```
alias.c checkout
```
```
alias.b branch
```
```
alias.s stash
```
```
alias.sa stash apply
```
```
alias.f fetch
```
```
alias.rh reset --hard
```
```
alias.rs reset --soft HEAD^
```
```
alias.sl stash list
```

# Branch list but in order of usage (easy to find recent branches)
```
alias.bc branch --sort=-committerdate
```

# Set upstream to current branchname in origin
```
alias.su !f() { git branch --set-upstream-to=origin/$(git rev-parse --abbrev-ref HEAD) $(git rev-parse --abbrev-ref HEAD); }; f
```

# Commands to remove local branches no longer needed. Scans currently checked out branch commits to see if each local branch commits are already present. 
The former acts as a dry run to let you know which branches will get deleted. The latter does so.
```
alias.cleanup !git branch --format '%(refname:short)' --merged | grep -E -v '(release.*|develop|^\*)' | xargs -n1 -r echo
```
```
alias.performcleanup !git branch --format '%(refname:short)' --merged | grep -E -v '(release.*|develop|^\*)' | xargs -n1 -r git branch -d
```

# Shorthand checkout and merges (if follow major release structure for branch names)
```
alias.c221 checkout release/22.1
```
```
alias.c231 checkout release/23.1
```
```
alias.c232 checkout release/23.2
```

```
alias.m221 merge origin/release/22.1
```
```
alias.m231 merge origin/release/23.1
```
```
alias.m232 merge origin/release/23.2
```
