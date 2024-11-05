Einbinden eines Kubernetes Secrets als einzelne Datei in einem Pod / Mounting a Kubernetes Secret as a single file inside a Pod

	15.	Januar 2019
Kürzlich musste ich einen privaten SSH-Schlüssel, der von einer App zur Verbindung mit einer anderen App verwendet wird, in einen laufenden Pod einbinden. Um dies sicherzustellen, haben wir den SSH-Schlüssel in ein Kubernetes Secret gelegt und dieses Secret dann in eine Datei im Pod-Spec für ein Deployment eingebunden.
Ich möchte den Prozess hier dokumentieren, weil (a) ich weiß, dass ich ihn wiederholen muss und es mir dadurch ein paar Minuten Recherche spart, und (b) es minimal unintuitiv ist (zumindest für mich).

Zuerst habe ich ein Secret in einem Namespace definiert:

apiVersion: v1  
kind: Secret  
metadata:  
  name: ssh-key  
  namespace: acme  
data:  
  id_rsa: {{ secret_value_base64_encoded }}  

Beachten Sie den Schlüssel id_rsa für die Secret-Daten – ich habe diesen gewählt, da beim Einbinden eines Secrets in ein Volume der Einhängepunkt ein Verzeichnis sein wird und jede Datei in diesem Verzeichnis einem Schlüssel in den Secret-Daten entspricht. In diesem Fall würde Kubernetes also eine Datei mit dem Namen id_rsa und dem Wert aus dem Secret im Verzeichnis /var/my-app platzieren, falls ich einen Mount-Pfad dorthin setze. (Hinweis: Ich verwende Ansible zur Template-Erstellung und Anwendung von Manifests. Daher nutze ich tatsächlich einen Wert wie {{ ansible_vault_encrypted_string | b64encode }}, was Ansible Vault zum Entschlüsseln eines verschlüsselten privaten Schlüssels in einer Playbook-Variable verwendet – das ist hier jedoch nebensächlich).

Um die Datei im Pfad /var/my-app/id_rsa einzubinden, füge ich das Volume wie folgt in meinem Deployment-Spec hinzu:

spec:  
  template:  
    spec:  
      containers:  
      - image: "my-image:latest"  
        name: my-app  
        ...  
        volumeMounts:  
          - mountPath: "/var/my-app"  
            name: ssh-key  
            readOnly: true  
      volumes:  
        - name: ssh-key  
          secret:  
            secretName: ssh-key  

Beachten Sie, dass Sie die Berechtigungen der Secret-Dateien mithilfe von defaultMode in der Volumes-Definition oder sogar individuell pro Datei steuern können (wenn es mehrere Schlüssel in den Secret-Daten gibt). Weitere Informationen dazu finden Sie in der Secrets-Dokumentation (insbesondere im Abschnitt über Berechtigungen von Secret-Dateien).

Ein Secret in einer einzelnen Datei in einem vorhandenen Verzeichnis einbinden / Mounting a secret to a single file in an existing directory

Eine Sache, die leider nicht unterstützt wird, ist das Einbinden eines einzelnen Secrets in eine einzelne Datei in einem Verzeichnis, das bereits im Container existiert. Dies bedeutet, dass Secrets nicht als Dateien im selben Verzeichnis eingebunden werden können, wie es bei einem „File-as-Volume-Mount“ in Docker oder dem Einbinden eines ConfigMap-Elements in ein vorhandenes Verzeichnis möglich ist. Wenn Sie ein Secret in ein Verzeichnis einbinden (wie im obigen Beispiel /var/my-app), wird Kubernetes das gesamte Verzeichnis /var/my-app nur mit den Inhalten Ihrer Secret-/secretName-Elemente füllen.

Um dieses Problem zu umgehen, können Sie das Secret an einem anderen Ort (z. B. /var/my-app-secrets) einbinden und dann einen postStart-Lifecycle-Hook verwenden, um es an die gewünschte Stelle zu kopieren:

        lifecycle:  
          postStart:  
            exec:  
              command:  
                - /bin/sh  
                - -c  
                - cp /var/my-app-secrets/id_rsa /var/my-app/id_rsa  

So bleiben die vorhandenen Inhalte des Verzeichnisses /var/my-app erhalten.

Weiterführende Lektüre / Further reading

	•	Cron-Jobs für Drupal in Kubernetes ausführen / Running Drupal Cron Jobs in Kubernetes
	•	Ein AWS EFS-Dateisystem auf einer EC2-Instanz mit Ansible einbinden / Mount an AWS EFS filesystem on an EC2 instance with Ansible
	•	php artisan schedule:run für Laravel in Kubernetes-CronJobs ausführen / Running ‘php artisan schedule:run’ for Laravel in Kubernetes CronJobs

Tags: secrets, kubernetes, manifest, k8sdeployment

Kommentare / Comments

Daniel – vor 5 Jahren / 5 years ago
Hallo, vielen Dank für das informative Tutorial.
Könnten Sie mehr Informationen dazu geben, wie Sie Ansible verwenden, um die Werte in der Secrets-Datei zu befüllen?
reply

Jack Replen – vor 5 Jahren / 5 years ago
Das war hilfreich, ich schaue hier regelmäßig vorbei, um mich daran zu erinnern, weil es der einfachste Leitfaden ist, den es gibt. Danke!
reply

Josiah – vor 5 Jahren / 5 years ago
In Linux erwarte ich, dass ein Mount alles im Verzeichnis unsichtbar macht. Hier scheint /var/my-app jedoch nicht gelöscht zu werden, sondern es wird hinzugefügt. Ist das korrekt?
reply

HenriTEL – vor 4 Jahren / 4 years ago
Seien Sie vorsichtig, wenn Sie Secrets in Containern verschieben, Secret-Volumes sind standardmäßig tmpfs, Sie könnten sie auf einem nichtflüchtigen Gerät schreiben. Es ist besser, im postStart-Hook einen Symlink zu verwenden.
reply

Pradeep – vor 4 Jahren / 4 years ago
Sie können subPath verwenden, um ein ConfigMap- oder Secret-Schlüssel in ein bestimmtes Unterverzeichnis einzubinden, ohne dessen Inhalt zu überschreiben.

volumeMounts:  
  - mountPath: "/var/my-app/id_rsa"  
    subPath: id_rsa  
    name: ssh-key  
    readOnly: true  
volumes:  
  - name: ssh-key  
    secret:  
      secretName: ssh-key  
      items:  
        - key: id_rsa  
          path: id_rsa  

reply

paws – vor 3 Jahren / 3 years ago
In Antwort auf „Sie können subPath verwenden“ von Pradeep
Danke, das hat für mich auf k8s v1.19 funktioniert. Jeff, vielleicht solltest du das bestätigen/deinen Artikel aktualisieren, scheint auch bei dir zu funktionieren.
reply

Xavi – vor 4 Jahren / 4 years ago
Sie können anstelle des Kopierens der Datei einen Link verwenden, so wird Ihre Datei bei Änderungen am Secret automatisch aktualisiert.

lifecycle:  
  postStart:  
    exec:  
      command:  
        - /bin/sh  
        - -c  
        - ln -sf /var/my-app-secrets/settings.json /var/my-app/settings.json  

reply

Mark – vor 4 Jahren / 4 years ago
Was passiert mit id_rsa beim Mounten? Wird es automatisch decodiert? Das ist mein Problem, es funktioniert für mich nicht. Wird das Decodieren manuell in einem separaten Befehl ausgeführt? Die k8s-Dokumentation sagt, dass es beim Mounten decodiert wird.
reply

Weitere Kommentare folgen mit ähnlichem Format …