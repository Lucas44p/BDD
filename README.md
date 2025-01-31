Partie 1 :  
Créer la base de données

```sql
CREATE DATABASE location_ski;
USE location_ski;


CREATE TABLE clients (
    noCli INT PRIMARY KEY,
    nom VARCHAR(30),
    prenom VARCHAR(30),
    adresse VARCHAR(120),
    cpo VARCHAR(5),
    ville VARCHAR(80)
);

CREATE TABLE fiches (
    noFic INT PRIMARY KEY,
    noCli INT,
    dateCrea DATE,
    datePaiement DATE,
    etat VARCHAR(2),
    FOREIGN KEY (noCli) REFERENCES clients(noCli)
);


CREATE TABLE tarifs (
    codeTarif CHAR(5) PRIMARY KEY,
    libelle CHAR(30),
    prixJour DECIMAL(5,0)
);


CREATE TABLE gammes (
    codeGam VARCHAR(5) PRIMARY KEY,
    libelle VARCHAR(45)
);


CREATE TABLE categories (
    codeCate VARCHAR(5) PRIMARY KEY,
    libelle VARCHAR(30)
);


CREATE TABLE grilleTarifs (
    codeGam VARCHAR(5),
    codeCate VARCHAR(5),
    codeTarif CHAR(5),
    PRIMARY KEY (codeGam, codeCate),
    FOREIGN KEY (codeGam) REFERENCES gammes(codeGam),
    FOREIGN KEY (codeCate) REFERENCES categories(codeCate),
    FOREIGN KEY (codeTarif) REFERENCES tarifs(codeTarif)
);


CREATE TABLE articles (
    refart VARCHAR(10) PRIMARY KEY,
    designation VARCHAR(100),
    codeGam CHAR(5),
    codeCate CHAR(5),
    FOREIGN KEY (codeGam) REFERENCES gammes(codeGam),
    FOREIGN KEY (codeCate) REFERENCES categories(codeCate)
);


CREATE TABLE lignesFic (
    noFic INT,
    noLig INT,
    refart CHAR(8),
    depart DATE,
    retour DATE,
    PRIMARY KEY (noFic, noLig),
    FOREIGN KEY (noFic) REFERENCES fiches(noFic),
    FOREIGN KEY (refart) REFERENCES articles(refart)
);
```

Partie 2 :
1️⃣ Liste des clients (toutes les informations) dont le nom commence par un D
```sql
SELECT * FROM clients WHERE nom LIKE 'D%';
```

2️⃣ Nom et prénom de tous les clients
```sql
SELECT prenom, nom FROM clients;
```

3️⃣ Liste des fiches (n°, état) pour les clients (nom, prénom) qui habitent en Loire Atlantique (44);
```sql
SELECT fiches.noFic, fiches.etat, clients.nom, clients.prenom
FROM fiches
JOIN clients ON fiches.noCli = clients.noCli
WHERE clients.cpo LIKE '44%';
```

4️⃣ Détail de la fiche n°1002
```sql
SELECT fiches.noFic, clients.nom, clients.prenom, articles.refart, articles.designation, lignesFic.depart, lignesFic.retour, tarifs.prixJour, 
(DATEDIFF(IFNULL(lignesFic.retour, NOW()), lignesFic.depart) * tarifs.prixJour) AS montant
FROM fiches
JOIN clients ON fiches.noCli = clients.noCli
JOIN lignesFic ON fiches.noFic = lignesFic.noFic
JOIN articles ON lignesFic.refart = articles.refart
JOIN grilleTarifs ON articles.codeGam = grilleTarifs.codeGam AND articles.codeCate = grilleTarifs.codeCate
JOIN tarifs ON grilleTarifs.codeTarif = tarifs.codeTarif
WHERE fiches.noFic = 1002;
```

5️⃣ Prix journalier moyen de location par gamme
```sql 
SELECT gammes.libelle AS Gamme, 
AVG(tarifs.prixJour) AS tarif_journalier_moyen FROM lignesFic
JOIN articles ON lignesFic.refart = articles.refart 
JOIN grilleTarifs ON articles.codeGam = grilleTarifs.codeGam  AND articles.codeCate = grilleTarifs.codeCate
JOIN tarifs ON grilleTarifs.codeTarif = tarifs.codeTarif    
JOIN gammes ON articles.codeGam = gammes.codeGam
GROUP BY gammes.libelle;
```


6️⃣ Détail de la fiche n°1002 avec le total
```sql
SELECT fiches.noFic, clients.nom, clients.prenom, articles.refart, articles.designation, lignesFic.depart, lignesFic.retour, tarifs.prixJour, 
(DATEDIFF(IFNULL(lignesFic.retour, NOW()), lignesFic.depart) * tarifs.prixJour) AS montant, 
SUM((DATEDIFF(COALESCE(lignesFic.retour, CURDATE()), lignesFic.depart) + IF(lignesFic.retour IS NOT NULL, 1, 0)) * tarifs.prixJour) OVER (PARTITION BY fiches.noFic) AS Total
FROM fiches
JOIN clients ON fiches.noCli = clients.noCli
JOIN lignesFic ON fiches.noFic = lignesFic.noFic
JOIN articles ON lignesFic.refart = articles.refart
JOIN grilleTarifs ON articles.codeGam = grilleTarifs.codeGam  AND articles.codeCate = grilleTarifs.codeCate         
JOIN tarifs ON grilleTarifs.codeTarif = tarifs.codeTarif
WHERE fiches.noFic = 1002;
```


7️⃣ Grille des tarifs
```sql
SELECT categories.libelle AS libelle, gammes.libelle AS Gamme, tarifs.libelle AS Tarif, tarifs.prixjour as tarifs
FROM grilleTarifs
JOIN gammes ON grilleTarifs.codeGam = gammes.codeGam
JOIN categories ON grilleTarifs.codeCate = categories.codeCate
JOIN tarifs on grilletarifs.codeTarif = tarifs.codeTarif;
```

8️⃣ Liste des locations de la catégorie SURF
```sql
SELECT articles.refart, articles.designation, COUNT(lignesFic.noFic) AS nbLocation
FROM articles
JOIN categories ON articles.codeCate = categories.codeCate
JOIN lignesFic ON articles.refart = lignesFic.refart
WHERE categories.libelle = 'SURF'
GROUP BY articles.refart, articles.designation;
```


9️⃣ Calcul du nombre moyen d’articles loués par fiche de location
```sql
SELECT AVG(nb_lignes) AS nb_lignes_moyen_par_fiche
FROM (
    SELECT fiches.noFic, COUNT(lignesFic.noLig) AS nb_lignes
    FROM fiches
    JOIN lignesFic ON fiches.noFic = lignesFic.noFic
    GROUP BY fiches.noFic
) AS sous_requete;
```


10 - Calcul du nombre de fiches de location établies pour les catégories de location Ski alpin, Surf et Patinette
```sql
SELECT categories.libelle as categories, COUNT(DISTINCT fiches.noFic) as nombre_de_location
FROM fiches
join lignesfic on fiches.noFic = lignesfic.noFic
JOIN articles ON lignesFic.refart = articles.refart
JOIN categories ON articles.codeCate = categories.codeCate
WHERE categories.libelle IN ('Ski alpin', 'Surf', 'Patinette')
GROUP BY categories.libelle;
```


11 Calcul du montant moyen des fiches de location
```sql
SELECT AVG(DATEDIFF(IFNULL(lignesFic.retour, NOW()), lignesFic.depart) * tarifs.prixJour) AS montant_moyen_fiche
FROM fiches
JOIN lignesFic ON fiches.noFic = lignesFic.noFic
JOIN articles ON lignesFic.refart = articles.refart
JOIN grilleTarifs ON articles.codeGam = grilleTarifs.codeGam AND articles.codeCate = grilleTarifs.codeCate
JOIN tarifs ON grilleTarifs.codeTarif = tarifs.codeTarif;
```