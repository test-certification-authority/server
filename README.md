# INSTALLAZIONE
di seguito  i link  per fare il download  di Open SSL:
# INSTALLAZIONE WINDOWS
https://tecadmin.net/install-openssl-on-windows/
# INSTALLAZIONE UNIX/LINUX/MACOS
https://github.com/openssl/openssl/blob/master/INSTALL.md#unix--linux--macos-1
# Creare un Server C.A. 
[In questo esempio è stato utilizzato un server con CentOS]

## Creare la struttura di directory della root C.A.
```bash
/root
|-- ca
|   |-- certs/
|   |-- crl/
|   |-- index.txt
|   |-- newcerts/
|   |-- openssl.cnf
|   |-- private/
|-- openssl.cnf
|   |-- serial
```

ca/certs -> conterrà i certificati della certification authority
ca/crl   -> conterrà le richieste di certification
ca/index.txt -> file che tiene traccia dei certificati firmati
ca/serial --> file che tiene traccia dei certificati firmati
ca/private/ -> contiene le chiavi private della c.a.
ca/openssl.cnf  -> Contiene le configurazioni di openssl, in questo modo openssl saprà come  comportarsi di fronte alle situazione di creazione di chiavi, certificati o revoca.

I comandi da usare per ottenere questa struttura sono

```bash
cd /root/
mkdir /root/ca

cd /root/ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
```

## Creare la chiave privata per la root CA
Come configurato nel file openssl.cnf il nome sarà ca.key.pem e verrà salvata nella directory private/
```bash
cd /root/ca
openssl genrsa -aes256 -out private/ca.key.pem 4096

# il terminale vi chiederà una passphrase per generare la vostra chiave privata
# successivamente vi chiederà di riscriverla a scopo di verifica
#adesso rendiamo inaccessibile questa chiave a utenti mailntenzionati
chmod 400 private/ca.key.pem

```

L'ultimo numero è il numero di bit per la chiave, viene scelto 4096 bit piuttosto che 2048 poichè la rootCA deve avere una robustezza e sicurezza maggiore poichè il suo certificato deve valere molti anni (anche 20 anni)


## Creare il certificato per la rootCA
```bash
cd /root/ca
openssl req -config openssl.cnf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem
```
Vi verrà chiesto di inserire la passphrase inserita in precedenza per generare la chiave privata e poi una serie di dati inerenti la rootCA.

Mettiamo in sicurezza anche il certificato
```bash
chmod 444 certs/ca.cert.pem
```
## Verifichiamo il certificato della rootCA
```bash 
openssl x509 -noout -text -in certs/ca.cert.pem
```

# Aggiungere una C.A. di secondo livello
Quando si parla in termini di C.A. si parla in termini gerarcici, in genere la rootCA non è esposta, ma bensì vengono creati gradi gerarchici di C.A. intermedie. In questo modo se venisse compromessa una di queste CA intermedie, la CA del livello superiore potrebbe revocare il certificato della CA compromessa e automaticamente questi verrebbero revocati.

## Creare la struttura di directory della C.A. intermedia


```bash
/root/ca/
|   |-- intermediate
|   |   |-- certs
|   |   |-- crl
|   |   |-- crlnumber
|   |   |-- csr
|   |   |-- index.txt
|   |   |-- openssl-intermediate.cnf
|   |   |-- newcerts
|   |   |-- openssl.cnf
|   |   |-- private
```

I comandi per aggiungere questa struttura sono

```bash
cd /root/
mkdir /root/ca

cd /root/ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
echo 1000 > /root/ca/intermediate/crlnumber
```` 

[crlnumber è utilizzato per tenere traccia della lista dei certificati revocati (certificate revocation list)]

##  Creare il file di configurazione ***openssl-intermerdiate.cnf***
```bash
cd /root/ca/intermediate
cp ../openssl.cnf ./openssl-intermediate.cnf 
```

nel file ***openssl-intermediate.cnf*** andranno modificati alcuni parametri come segue
```bash
[ CA_default ]
dir             = /root/ca/intermediate
private_key     = $dir/private/intermediate.key.pem
certificate     = $dir/certs/intermediate.cert.pem
crl             = $dir/crl/intermediate.crl.pem
policy          = policy_loose
```
## Generare la chiave privata della CA intermedia
Come fatto in precedenza
```bash 
cd root/ca
openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
# Vi verrà chiesto di creare la passphrase  per la chiave privata della CA intermedia
chmod 400 intermediate/private/intermediate.key.pem
```
## Generare il certificato della CA intermedia
[il file di configurazione usato è intermediate/openssl.cnf]
```bash
cd /root/ca  openssl req -config intermediate/openssl.cnf -new -sha256  -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem
```
Verranno richieste informazioni della CA intermedia che poi saranno incorporate nel certificato.

### Firmiamo la richiesta di certificazione, della CA intermedia, dalla rootCA

```bash
cd /root/ca
openssl ca -config openssl.cnf -extensions v3_intermediate_ca   -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem  -out intermediate/certs/intermediate.cert.pem
# Verrà richiesto di inserire la passphrase della rootCA e poi se si intende firmare il certificato
#Mettiamo in sicurezza il certificato ottenuto
chmod 444 intermediate/certs/intermediate.cert.pem
```
### Creare la catena dei certificati
```bash
cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

creare la chiave pubblica e privata, certificato autofirmato per la certification authority  (in questo caso valido 365 giorni)

Sostituire il valore dell'argomento -subj con i propri dati

```bash
openssl req -config ./openssl-ca.cnf -new -x509 -days 365 -sha256 -newkey rsa:2048 -keyout ./options/ca/private/ca.key -subj "/C=IT/ST=Italy/O=Test Certification Authority/CN=www.test-certification-authority.it" -out ./options/ca/ca.crt
```
Inserire la password e la verifica della password 
**NB: Questa password andrà ricordata e applicata ogni qualvolta la C.A. dovrà firmare certificato**

**Da Aggiungere schermata e spiegazione dei dati su -subj**


Cambiare i permessi di accesso alla chiave 
```bash
chmod 700 ./options/ca
chmod 600 ./options/ca/private/ca.key
```



# CREARE UN CERTIFICATO FIRMATO DALLA C.A.
Prendere la chiave inserita dall'utente e posizionarla nella directory delle richieste (valido 90 giorni)

# Firmare il certificato per l'utente
A questo punto il server C.A. è pronto per iniziare a creare i primi certificati
L'utente inizialmente dovrà creare la propria chiave privata e caricarla sul server.

## CREARE UNA CHIAVE PRIVATA [UTENTE]
sostituire cognome.nome.matricola in minuscolo con i rispettivi dati senza spazi, accenti, apostrofi o altri caratteri speciali (altrimenti genera errore)

*es1 mario.rossi.000001* = Mario Rossi 000001
*es2 mario.pasto.000002* = Mario Pastò 000002
*es3 fra.martin.000003* = Fra' Martino 000003

```bash
openssl genrsa -out cognome.nome.matricola.key.pem 2048
```

sostituire *matricola* con la propria matricola
*es -out 000001.csr.pem*

```bash
openssl req -new -sha256 -key cognome.nome.matricola.key.pem -subj "/C=IT/ST=Italy/O=UniPD/OU=Studente/CN=cognome.nome.matricola" -out matricola.csr.pem
```
Abbiamo ottenuto due file
cognome.nome.matricola.key.pem <-- la nostra chiave privata
matricola.csr.pem <--- il file da caricare sul server per ricevere il certificato

[l'interfaccia che si occuperà del caricamento del file .csr.pem dovrà posizionare tale file nella directory
intermediate/csr/]

### FIRMARE E CREARE IL CERTIFICATO
```bash
cd /root/ca 
openssl ca -config intermediate/openssl.cnf -extensions server_cert -days 375 -notext -md sha256   -in intermediate/csr/matricola.csr.pem -out intermediate/certs/matricola.cert.pem chmod 444 intermediate/certs/matricola.cert.pem
```

### VERIFICARE IL CONTENUTO DEL CERTIFICATO
```bash 
openssl x509 -noout -text \
      -in intermediate/certs/matricola.cert.pem
```

# Certification Revocation List
## preparare il file di configurazione
Modficare il file intermediate/openssl.cnf
modificando il parametro crlDistributionPoints come segue
```bash
  [ server_cert ]
crlDistributionPoints = URI:http://nostroDominio.ext/intermediate.crl.pem
```

## Creare la CRL
```bash
cd /root/ca
openssl ca -config intermediate/openssl.cnf -gencrl -out intermediate/crl/intermediate.crl.pem

#puoi controllare il contenuto della crl con il comando
openssl crl -in intermediate/crl/intermediate.crl.pem -noout -text
```
## Revocare un certificato
```bash
openssl ca -config intermediate/openssl.cnf -revoke intermediate/certs/certificatoDaRevocare.cert.pem
```

# Convertire un certificato .pem in altri formati [UTENTE]
```bash
openssl x509 -outform der -in nomeCertificato.pem -out nomeCertificato.der
```
### Convert PEM to P7B
```bash
openssl crl2pkcs7 -nocrl -certfile nomeCertificato.cer -out nomeCertificato.p7b -certfile CACert.cer
```
### Convert PEM to PFX
```bash
openssl pkcs12 -export -out nomeCertificato.pfx -inkey privateKey.key -in nomeCertificato.crt -certfile CACert.crt
```

#	TEST

```bash
#creare la chiave pubblica e privata, certificato autofirmato per la certification authority
#UTENTE
cd %UserProfile%\Desktop
##creo la chiave privata per l'utente
openssl genrsa -out rossi.mario.0000001.key 2048
##creo la certification request da parte dell'utente
openssl req -new -sha256 -key rossi.mario.0000001.key -subj "/C=IT/ST=Italy/O=UniPD/OU=Studente/CN=rossi.mario.0000001" -out 0000001.csr.pem
```
Lo studente ora caricherà il file .csr.pem nell'interfaccia online per la generazione del certificato
e otterrà il suo certificato .cert.pem

Per utilizzarlo per firmare i file dovrà convertirlo in relazione al formato richiesto dal programma di firma


