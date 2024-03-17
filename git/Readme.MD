# Multiple accounts setup on same workstation
1. Generate and set private and public keys for both accounts (suggested structure for git keys `~/.ssh/git/`)
2. Optional - setup `~/.ssh/config`
```shell
Host github-home
    HostName github.com
    User git
    IdentityFile ~/.ssh/git/id_rsa_home

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/git/id_rsa_work
```
3. Setup git aliases `git config --global --edit`
```shell
[alias]
        home-clone = !git -c core.sshCommand='ssh -i ~/.ssh/git/home' clone
        home-init = !git -c core.sshCommand='ssh -i ~/.ssh/git/home' init
        work-clone = !git -c core.sshCommand='ssh -i ~/.ssh/git/work' clone
        work-init = !git -c core.sshCommand='ssh -i ~/.ssh/git/work' init
```
4. You may need to add a key: `ssh-add ~/.ssh/github_keys/git-home`
5. To push to a new repo - setup remote url: `git remote set-url origin git@github-home:dmitry87/manuals.git` <br \>
Where `github-home` is host from 2  <br \>
`dmitry87` - account name  <br \>
`manuals` - name of the repo (you should create it on server first)
6. To setup username for the repo you should add following settings:
```shell
git config user.name "Your Name Here"
git config user.email "your@email.example"
```

## Alternative 
1. install tool
`npm install -g git-profile`
2. Create profiles:
```shell
git-profile add home
git-profile add work
```
3. Configure Profiles:
```shell
git-profile set home user.name "Your Home Name"
git-profile set home user.email "home@example.com"

git-profile set work user.name "Your Work Name"
git-profile set work user.email "work@example.com"
```
4. Switch between profiles
```shell
git-profile use home
# Now your Git commands will use the home account

git-profile use work
# Now your Git commands will use the work account

```