# opsh — Mini-shell en Python

Projet de **Systèmes d'Exploitation** (UIR/ESIN). `opsh` est un interpréteur de
commandes écrit en Python qui reprend les mécanismes réels de Bash : création de
processus (`fork` / `exec` / `wait`), redirections, pipes, signaux, variables,
substitution de commandes et scripts.

> Les commandes sont **renommées** par rapport à Bash, et le pipe `|` devient `->`.

## Auteurs

Groupe :

- Marwa Bechnikha
- Dina Jalal
- Zakaria Nasroallah
- Mohamed Reda Elrhoul

UIR/ESIN — Systèmes d'Exploitation

## Prérequis

- Linux, macOS ou WSL sous Windows (le shell utilise `os.fork`, `os.pipe`, `os.dup2`)
- Python 3.9 ou plus récent
- Aucune dépendance externe (bibliothèque standard uniquement)

## Installation et lancement

```bash
git clone https://github.com/Shellcore-UIR/opsh.git
cd opsh
chmod +x opsh
python3 opsh              # mode interactif (REPL)
python3 opsh script.opsh  # mode script
```

## Builtins

| Bash | opsh | Description |
|------|------|-------------|
| `cd` | `go [dir]` | Change de répertoire (`go -` revient au précédent) |
| `ls` | `list [-a] [-l] [chemin]` | Liste un répertoire |
| `cat` | `show [fichier]` | Affiche un fichier (sans argument : lit l'entrée standard) |
| `pwd` | `path` | Affiche le répertoire courant |
| `echo` | `say [-n] texte` | Affiche du texte |
| `cp` | `copy [-r] src dst` | Copie |
| `mv` | `move src dst` | Déplace ou renomme |
| `rm` | `del [-r] [-f] cible` | Supprime |
| `mkdir` | `newdir [-p] dossier` | Crée un dossier |
| `touch` | `newfile fichier` | Crée un fichier ou met à jour sa date |
| `history` | `past` | Affiche l'historique |
| `help` | `info` | Affiche l'aide |
| `exit` | `quit [code]` | Quitte le shell |
| `\|` | `->` | Pipe |

Builtins supplémentaires : `osinfo` (affiche `Systèmes d’Exploitation –- UIR/ESIN`),
`export`, `unset`, `jobs`, `test`.

Toute autre commande (`wc`, `ps`, `sleep`, `grep`...) est cherchée dans le `PATH`
et lancée via `fork` + `execvp`.

## Syntaxe supportée

### Redirections

```
say hello > f.txt          # écrase f.txt
say world >> f.txt         # ajoute à la fin
wc -l < f.txt              # lit l'entrée depuis un fichier
```

### Pipes (`->`)

```
show f.txt -> wc -l
list -> sort -> head -3
```

### Arrière-plan et signaux

```
sleep 10 &                 # affiche [pid], le shell reste utilisable
jobs                       # liste les processus en arrière-plan
```

- **Ctrl-C** (SIGINT) interrompt la commande en cours, pas le shell.
- **SIGCHLD** : le shell récolte les processus d'arrière-plan terminés
  (pas de zombies) et affiche leur statut.

### Variables et expansion

```
nom=Alice                  # variable du shell
export PATH=$PATH:/opt     # variable d'environnement
say ${nom} $?              # $? = statut de la dernière commande
sleep 5 &
say $!                     # $! = PID de la dernière commande en arrière-plan
say $$                     # PID du shell
```

Les quotes simples `'...'` désactivent l'expansion ; les doubles `"..."` la permettent.

### Substitution de commandes

```
say "Dossier : $(path)"
d=`path`
```

### Fichier de configuration

Au démarrage en mode interactif, `~/.opshrc` est exécuté ligne par ligne :

```
export EDITOR=nano
say Bienvenue dans opsh
```

### Historique et complétion

- Historique persistant dans `~/.opsh_history`, flèches haut/bas, commande `past`.
- Touche **Tab** : complète les commandes (builtins et `PATH`) et les fichiers.

### Mode script

```
# exemple.opsh
for x in a b c
do
  say item $x
done

n=0
while test $n -lt 3
do
  say n=$n
  n=$(expr $n + 1)
done

if test -f opsh
  say le fichier existe
else
  say introuvable
fi
```

Une condition est une commande : elle est vraie si son statut vaut 0.
Les mots `then` et `do` sont optionnels ; chaque bloc se termine par `fi` ou `done`.

## Architecture du code

| Fonction | Rôle |
|----------|------|
| `tokenize()` | Analyse lexicale : quotes, `$VAR`, `$(...)`, backticks, opérateurs |
| `parse()` | Découpe par `->`, extrait les redirections et le `&` |
| `execute()` | Crée les processus (`fork`), branche les pipes (`pipe` + `dup2`), attend (`waitpid`) |
| `child_main()` | Code du fils : redirections, puis builtin ou `execvp` |
| `run_builtin_inprocess()` | Builtin seul exécuté dans le shell (indispensable pour `go`) |
| `capture()` | Substitution de commandes : fork + pipe, le père lit la sortie du fils |
| `on_sigchld()` | Récolte des processus d'arrière-plan |
| `parse_block()` / `exec_nodes()` | Mode script : `if`, `for`, `while` |

## Concepts illustrés

- **Programme vs processus** : un programme est un fichier ; un processus est son
  exécution (PID, mémoire, descripteurs).
- **fork / exec / wait** : `fork` duplique le processus, `exec` remplace son image
  par un autre programme, `wait` récupère son statut.
- **Zombie / orphelin** : un fils terminé non attendu reste zombie ; un fils dont le
  père meurt est adopté par `init`.
- **Descripteurs 0/1/2** : stdin, stdout, stderr ; une redirection est un `dup2`
  effectué avant `exec`.
- **Pipe = IPC** : tampon du noyau entre deux processus ; il faut fermer les
  extrémités inutiles pour que le lecteur reçoive la fin de fichier.
- **Signaux** : SIGINT (Ctrl-C), SIGCHLD (fils terminé).
- **Pourquoi `go` est un builtin** : le répertoire courant est un attribut du
  processus ; un fils ne peut pas modifier celui de son père.
- **Mode utilisateur / noyau** : `fork`, `execve`, `pipe`, `dup2`, `waitpid` sont des
  appels système qui font passer le processus en mode noyau.
- **PATH / environnement** : `execvp` cherche l'exécutable dans les dossiers de
  `PATH` ; l'environnement est hérité par les fils.

## Démonstration

```
opsh> list
opsh> go ..
opsh> path
opsh> say hello > f.txt
opsh> show f.txt
opsh> show f.txt -> wc -l
opsh> sleep 10 &
opsh> ps
opsh> osinfo
```

## Limitations connues

- Pas de `;`, `&&`, `||` ni de `2>` / `2>&1`.
- Pas de développement des jokers (`*`, `?`) ; la complétion Tab couvre les fichiers.
- Pas de découpage en mots après expansion de variable.
- Pas de job control complet (`fg` / `bg`) : seul `jobs` est fourni.
- Les conditions de script utilisent le builtin `test` ; pas d'arithmétique intégrée
  (utiliser `expr`).
