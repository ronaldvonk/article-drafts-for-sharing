# VSCode setup

## Multiple git accounts:

https://stackoverflow.com/a/79446430

```bash
gh auth switch
```

Set the global git config to the user.email and name that use the most, then configure the username and email for a specific repository if you want to override that (do that before creating commits etc!):

```bash
git config --local user.email "your@email.com"
git config --local user.name "Your Name"
```

You can also use an extension on VSCode to manage your configs. eg: git-autoconfig
