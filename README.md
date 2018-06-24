# Projekt na egzamin - zespół *Anomalia*

**Członkowie zespołu**

 - Amelia Błachuciak [(zaliczenie)](https://github.com/erathiel/nosql)
 - Marta Walczak [(zaliczenie)](https://github.com/mawala/projekty-nosql)
 
## Główny projekt znajduje się **[na stronie](https://mawalca.github.io/egzamin-nosql/)**.
 
### Przedstawienie zbioru danych

Zbiór danych: [Zapytania i tagi ze Stack Overflow](https://github.com/dgrtwo/StackLite) (z dnia 7.04.2017)
 
Zbiór podzielony jest na 2 pliki *csv*:

 1. **Plik questions.csv** posiada 17 763 486 rekordów oraz zajmuje około 890 MB.

```bash
{
        "Id" : 85,
        "CreationDate" : "2008-08-01T14:19:52Z",
        "ClosedDate" : "2015-10-22T13:43:51Z",
        "DeletionDate" : "NA",
        "Score" : 89,
        "OwnerUserId" : 59,
        "AnswerCount" : 12
}
```

 - Id - id pytania
 - CreationDate - data utworzenia pytania
 - ClosedDate - data zamknięcia pytania lub "NA" w przypadku braku zamknięcia
 - DeletionDate - data usunięcia pytania lub "NA" w przypadku ciągłego istnienia
 - Score - wewnętrzny system oceniania użytkowników StackOverflow
 - OwnerUserId - id użytkownika, który umieścił dane pytanie
 - AnswerCount - ilość odpowiedzi udzielonych na dane pytanie

 2. **Plik question_tags.csv** posiada 52 224 835 rekordów oraz zajmuje około 871 MB.

```bash
{ "Id" : 85, "Tag" : "php" }
{ "Id" : 85, "Tag" : "sql" }
{ "Id" : 85, "Tag" : "database" }
{ "Id" : 85, "Tag" : "flat-file" }
```

- Id - identyfikator pytania z pliku questions.csv
- Tag - dany tag pytania
 
### Import danych
 
 1. questions.csv
 
```bash
powershell "Measure-Command{mongoimport -d anomalia -c questions --type csv --file questions.csv --headerline --drop}"
```

| Czas importu | Liczba rekordów |
|--------------|-----------------|
| 4,90 min     | 17 763 486      |

 2. question_tags.csv

```bash
powershell "Measure-Command{mongoimport -d anomalia -c question_tags --type csv --file question_tags.csv --headerline --drop}"
```

| Czas importu | Liczba rekordów |
|--------------|-----------------|
| 9,58 min     | 52 224 835      |

### Wstępna obróbka danych
 
 W celu wykorzystania możliwości MongoDB dane z obu powyżej opisanych plików zostaną zapisane do jednej kolekcji.
 
Chcemy uzyskać dokumenty postaci
 
```bash
{
        "_id" : ObjectId("58f528a58e05fcfb5031094a"),
        "Id" : 85,
        "CreationDate" : "2008-08-01T14:19:52Z",
        "ClosedDate" : "2015-10-22T13:43:51Z",
        "DeletionDate" : "NA",
        "Score" : 89,
        "OwnerUserId" : 59,
        "AnswerCount" : 12,
        "Tags" : [
                "php",
                "sql",
                "database",
                "flat-file"
        ]
}
```

Wykonamy więc agregację.

#### Etapy agregacji

- Etap *stage1* łączy kolekcję agregowaną (którą będzie kolekcja *questions*) z kolekcją *question_tags* po polach *Id* z obu kolekcji

```bash
stage1 =
{
  $lookup: {
    from: "question_tags",
    localField: "Id",
    foreignField: "Id",
    as: "Tags"
  }
}
```

W jego efekcie otrzymamy dokumenty w postaci

```bash
{
        "_id" : ObjectId("58f683bd5d98cb55b9d0819c"),
        "Id" : 85,
        "CreationDate" : "2008-08-01T14:19:52Z",
        "ClosedDate" : "2015-10-22T13:43:51Z",
        "DeletionDate" : "NA",
        "Score" : 89,
        "OwnerUserId" : 59,
        "AnswerCount" : 12,
        "Tags" : [
                {
                        "_id" : ObjectId("58f685905d98cb55b9dfd402"),
                        "Id" : 85,
                        "Tag" : "php"
                },
                {
                        "_id" : ObjectId("58f685905d98cb55b9dfd403"),
                        "Id" : 85,
                        "Tag" : "sql"
                },
                {
                        "_id" : ObjectId("58f685905d98cb55b9dfd404"),
                        "Id" : 85,
                        "Tag" : "database"
                },
                {
                        "_id" : ObjectId("58f685905d98cb55b9dfd405"),
                        "Id" : 85,
                        "Tag" : "flat-file"
                }
        ]
}
```

 - Etap *stage2* ze swej definicji powinien dodawać nowe pole. Jednak, ponieważ dokumenty już zawierają pole o takiej nazwie,
 więc etap ten zamienia istniejące pole *Tags*
 
```bash
stage2 =
{
  $addFields: {
    "Tags": "$Tags.Tag"  
  }
}
```

Dzięki temu etapowi otrzymujemy oczekiwany wygląd dokumentów

```bash
{
        "_id" : ObjectId("58f683bd5d98cb55b9d0819c"),
        "Id" : 85,
        "CreationDate" : "2008-08-01T14:19:52Z",
        "ClosedDate" : "2015-10-22T13:43:51Z",
        "DeletionDate" : "NA",
        "Score" : 89,
        "OwnerUserId" : 59,
        "AnswerCount" : 12,
        "Tags" : [
                "php",
                "sql",
                "database",
                "flat-file"
        ]
}
```

 - Etap *stage3* zapisuje otrzymane dokumenty do kolekcji o nazwie *stack*
 
```bash
stage3 =
{
  $out: "stack"
}
```

#### Optymalizacja agregracji

By przyspieszyć powyższe etapy agregracji (a głównie *stage1*), przed wykonaniem agregracji dodamy indeksy do obu kolekcji
*questions* i *question_tags* na polach *Id*

```bash
db.questions.createIndex( { "Id": 1 } )
db.question_tags.createIndex( { "Id": 1 } )
```

Czas dodania tych indeksów wynosi odpowiednio 48s oraz 135s.

#### Wykonanie agregracji

Agregację wykonujemy na kolekcji *questions*

```bash
db.questions.aggregate( [ stage1, stage2, stage3 ] )
```
