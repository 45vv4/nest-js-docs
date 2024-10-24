Gerne erkläre ich dir die Begriffe, die auf dem Bild zu sehen sind, und gebe dir jeweils ein Beispiel für jeden Begriff im Zusammenhang mit Schema-Design Anti-Patterns:

	1.	Massive Arrays:
	•	Erklärung: Ein Anti-Pattern ist es, große Arrays innerhalb eines Dokuments zu speichern. Wenn ein Array sehr groß wird, kann dies die Lese- und Schreiboperationen negativ beeinflussen, da bei jeder Abfrage das gesamte Dokument gelesen werden muss.
	•	Beispiel: Eine Datenbank speichert Benutzerprofile, und jedes Profil enthält ein Array aller bisherigen Bestellungen eines Benutzers. Bei einem Benutzer, der tausende Bestellungen hat, kann dieses Array sehr groß werden, was zu Performanceproblemen führt.
	2.	Massive Number of Collections:
	•	Erklärung: Zu viele Sammlungen (Collections) können die Verwaltung und die Leistung einer Datenbank erschweren. Es kann auch die Abfrageeffizienz beeinträchtigen.
	•	Beispiel: Wenn für jeden Monat eine neue Sammlung für Transaktionsdaten erstellt wird, anstatt alle Transaktionen in einer einzigen Sammlung zu speichern und nach dem Datum zu filtern, führt dies zu einer unnötigen Aufblähung der Datenbankstruktur.
	3.	Unnecessary Indexes:
	•	Erklärung: Zu viele oder unnötige Indizes verlangsamen die Leistung, da jede Änderung an der Datenbank zu einer Aktualisierung aller Indizes führt. Indizes sollten nur für Felder erstellt werden, die oft abgefragt werden.
	•	Beispiel: Eine E-Commerce-Datenbank hat Indizes für alle Felder eines Produktdokuments (Name, Preis, Beschreibung usw.), obwohl nur nach dem Produktnamen und der Kategorie gesucht wird.
	4.	Bloated Documents:
	•	Erklärung: Aufgeblähte Dokumente entstehen, wenn Dokumente zu viele Felder oder unnötige Daten enthalten. Dies führt zu Speicher- und Performanceproblemen.
	•	Beispiel: Ein Dokument, das einen Artikel beschreibt, enthält detaillierte Informationen, die selten gebraucht werden, wie historische Preise oder benutzerdefinierte Bewertungen. Diese könnten in separate Sammlungen ausgelagert werden, anstatt das Hauptdokument unnötig zu vergrößern.
	5.	Case-insensitive Queries without Case-insensitive Indexes:
	•	Erklärung: Wenn Anfragen nicht zwischen Groß- und Kleinschreibung unterscheiden, aber keine case-insensitive Indizes verwendet werden, müssen alle Daten durchsucht werden, was die Abfrage verlangsamt.
	•	Beispiel: Eine Abfrage sucht nach einem Benutzernamen unabhängig von der Groß- oder Kleinschreibung, aber es gibt keinen passenden Index. Dadurch wird die gesamte Sammlung durchlaufen, was ineffizient ist.
	6.	Separating Data That Is Accessed Together:
	•	Erklärung: Daten, die häufig gemeinsam abgefragt werden, sollten in einem Dokument zusammen gespeichert werden. Wenn sie getrennt gespeichert werden, sind zusätzliche Joins oder Abfragen erforderlich, was die Leistung beeinträchtigen kann.
	•	Beispiel: Kunden- und Bestelldaten werden in separaten Sammlungen gespeichert, obwohl sie oft zusammen abgefragt werden. Es wäre effizienter, die Bestellungen direkt im Kundendokument einzubetten.

Jedes dieser Anti-Patterns kann zu einer schlechteren Performance oder komplizierteren Abfragen führen. Es ist wichtig, bei der Modellierung von Datenbanken die richtige Balance zwischen Speicherplatz und Performance zu finden.