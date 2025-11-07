
### Alle remote repositories listen
git remote -v

### alle lokalen tags listen
git tag
### alle remote tags listen
git ls-remote --tags githubmain

### lokales tag löschen
git tag -d pre1.1.0.8

### tag auf remote löschen
git push origin --delete pre1.1.0.8
git push githubmain --delete pre1.1.0.8

### branch erstellen und in den branch wechseln
git switch -c newbranch

### branch erstellen
git branch newbranch

### in branch wechseln
git switch newbranch

### main in den branch mergen
git merge main // im branch ausführen


