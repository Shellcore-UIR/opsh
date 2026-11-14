# opsh — Mini-shell en Python

Projet de **Systèmes d'Exploitation** (UIR/ESIN). `opsh` est un interpréteur de
commandes écrit en Python qui reprend les mécanismes réels de Bash : création de
processus (`fork` / `exec` / `wait`), redirections, pipes, signaux, variables,
substitution de commandes et scripts.

> Les commandes sont **renommées** par rapport à Bash, et le pipe `|` devient `->`.

## Prérequis

- Linux, macOS ou WSL sous Windows (le shell utilise `os.fork`, `os.pipe`, `os.dup2`)
- Python 3.9 ou plus récent
- Aucune dépendance externe

## Lancement

```bash
chmod +x opsh
python3 opsh              # mode interactif
python3 opsh script.opsh  # mode script
```

## Builtins

| Bash | opsh | Description |
|------|------|-------------|
| `cd` | `go [dir]` | Change de répertoire |
| `ls` | `list [-a] [-l]` | Liste un répertoire |
| `cat` | `show [fichier]` | Affiche un fichier (ou stdin) |
| `pwd` | `path` | Répertoire courant |
| `echo` | `say [-n] texte` | Affiche du texte |
| `cp` | `copy [-r] src dst` | Copie |
| `mv` | `move src dst` | Déplace ou renomme |
| `rm` | `del [-r] [-f] cible` | Supprime |
| `mkdir` | `newdir [-p] dossier` | Crée un dossier |
| `touch` | `newfile fichier` | Crée un fichier |
| `history` | `past` | Historique |
| `help` | `info` | Aide |
| `exit` | `quit [code]` | Quitte |
| `\|` | `->` | Pipe |

Builtins supplémentaires : `osinfo`, `export`, `unset`, `jobs`, `test`.
Les autres commandes (`wc`, `ps`, `sleep`...) sont cherchées dans le `PATH`.

## Syntaxe

```
say hello > f.txt          # redirection (écrase)
say world >> f.txt         # redirection (ajoute)
wc -l < f.txt              # entrée depuis un fichier
show f.txt -> wc -l        # pipe
sleep 10 &                 # arrière-plan
x=42                       # variable
export PATH=$PATH:/opt     # variable d'environnement
say ${x} $? $!             # expansion
say $(path)                # substitution de commande
```

## Mode script

```
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
  say existe
else
  say absent
fi
```

## Architecture

| Fonction | Rôle |
|----------|------|
| `tokenize()` | Analyse lexicale (quotes, `$VAR`, `$(...)`, opérateurs) |
| `parse()` | Découpe par `->`, extrait redirections et `&` |
| `execute()` | `fork`, `pipe`, `dup2`, `waitpid` |
| `child_main()` | Code du fils : redirections puis `execvp` |
| `capture()` | Substitution de commandes (fork + pipe) |
| `on_sigchld()` | Récolte des processus d'arrière-plan |
| `parse_block()` | Mode script : `if`, `for`, `while` |

## Concepts illustrés

- **fork / exec / wait** : duplication d'un processus, remplacement de son image, récupération du statut.
- **Zombie / orphelin** : un fils terminé non attendu reste zombie (évité via SIGCHLD).
- **Redirections** : `dup2` sur les descripteurs 0, 1, 2 avant `exec`.
- **Pipe** : IPC entre deux processus ; il faut fermer les extrémités inutiles.
- **Signaux** : SIGINT (Ctrl-C) n'interrompt pas le shell, SIGCHLD récolte les fils.
- **`go` est un builtin** : le répertoire courant appartient au processus ; un fils ne peut pas modifier celui du shell.
- **Mode utilisateur / noyau** : `fork`, `execve`, `pipe`, `dup2`, `waitpid` sont des appels système.
- **PATH / environnement** : `execvp` cherche dans `PATH` ; l'environnement est hérité par les fils.

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

## Limitations

- Pas de `;`, `&&`, `||`, ni de `2>`.
- Pas de développement des jokers (`*`, `?`).
- Pas de job control complet (`fg`/`bg`), seulement `jobs`.

## Auteur

Marwa — UIR/ESIN — Systèmes d'Exploitation
