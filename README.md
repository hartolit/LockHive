# LockHive

LockHive er en password manager udviklet som et skoleprojekt på ZBC Ringsted. Projektet består af en .NET microservice backend og en Flutter frontend.

**Udviklere:** Daniel Vuust, Rasmus Thougaard, Benjamin Hoffmeyer.

## Arkitektur og Designvalg

Systemet er bygget op omkring en microservice-arkitektur for at sikre "Separation of Concerns". Hvert ansvarsområde (Users, Passwords, PaymentCards, KeyVaults) kører som isolerede services.

### Backend (.NET)
Backenden benytter en **Clean Architecture** tilgang.

* **Azure Service Bus (Rebus):** Vi bruger en service bus til at afkoble vores services. Når en bruger f.eks. opretter et password, sendes en command (`CreateUserPasswordCommand`) til bussen. Dette sikrer:
    * At services ikke behøver kende hinanden direkte.
    * At vi kan buffere requests, hvis en service er midlertidigt nede.
* **Command Pattern:** Alle skrivende handlinger er indkapslet i Commands. Dette adskiller dataen (hvad der skal gøres) fra udførslen (hvordan det gøres).
* **Operations:** Da kommunikationen via Service Bus er asynkron, bruger vi en "Operation" entitet. Frontend får et Operation ID tilbage med det samme og kan polle status på anmodningen, mens den behandles i baggrunden.

### Frontend (Flutter)
Mobilapplikationen er udviklet i Flutter med fokus på streng state management.

* **BLoC (Business Logic Component):** Styrer logikken og tilstanden i UI'en via Streams (Events ind -> States ud).
* **Provider:** Bruges til Dependency Injection igennem widget-træet.

## Password Generering (OWASP & NIST Compliance)

En af kernefunktionerne er generering af sikre kodeord. Implementationen følger retningslinjer fra OWASP og NIST for at sikre høj entropi og kompleksitet.

Logikken findes i `GenerateSecureKeyService.cs` og overholder følgende:

* **Kryptografisk Sikkerhed (CSPRNG):**
  Vi benytter `System.Security.Cryptography.RandomNumberGenerator` i stedet for standard `Random` klassen. Dette sikrer, at tallene er kryptografisk sikre og ikke kan forudsiges.
* **Kompleksitetskrav:**
  Algoritmen tvinger inklusion af karakterer fra fire forskellige sæt for at maksimere styrken:
  1. Store bogstaver (A-Z)
  2. Små bogstaver (a-z)
  3. Tal (0-9)
  4. Specialtegn (`!@#$%^&*...`)
  
  Koden genererer først en tilfældig streng og overskriver derefter specifikt tilfældige positioner med mindst én karakter fra hver kategori for at garantere overholdelse af kravene.
* **Længdebegrænsninger:**
  * **Min. 8 tegn:** Håndhæves for at overholde NIST minimumskrav.
  * **Maks. 128 tegn:** Sat for at beskytte systemet mod performance-problemer og potentielle angreb, hvor ekstremt lange strenge kunne udnyttes.

## Sikkerhedsanalyse & Roadmap

Som en del af projektet har vi arbejdet med kryptografi og analyseret forskellige krypteringstilstande (Modes of Operation).

### Nuværende Implementering: AES ECB
Systemet kører i øjeblikket med **AES ECB** (Electronic Codebook).
* **Virkemåde:** Hver 16-byte blok af data krypteres uafhængigt med samme nøgle.
* **Kendt Sårbarhed:** ECB er deterministisk. Det betyder, at to ens blokke af klartekst resulterer i ens ciphertext. I praksis kan dette afsløre mønstre i dataen (visualiseret ved den klassiske "Linux Pingvin" demo).

### Analyse af Alternativer (CBC vs. CTR)
Vi undersøgte **AES CBC** (Cipher Block Chaining) som en løsning på ECB's mønster-problem, da CBC bruger en Initialization Vector (IV) til at randomisere outputtet.
* **Hvorfor vi forkastede CBC:** Selvom CBC skjuler mønstre, introducerer det en afhængighed mellem blokkene (chaining). Dette gør, at kryptering ikke kan paralleliseres, hvilket kan blive en flaskehals ved store mængder data.

### Fremtidig Løsning: AES CTR
Vores analyse peger på, at den optimale løsning til en fremtidig iteration ville være **AES CTR** (Counter Mode) eller en moderne variant som **GCM** (Galois/Counter Mode).
* **Fordele ved CTR:**
    * **Parallelisering:** Da CTR fungerer som en stream cipher ved at kryptere en tæller (counter) i stedet for dataen direkte, kan både kryptering og dekryptering ske parallelt på tværs af flere CPU-kerner.
    * **Ingen Padding:** CTR kræver ikke padding af dataen, hvilket simplificerer implementeringen ift. CBC.
    * **Sikkerhed:** Løser ECB's deterministiske problem ved at hver blok bruger en unik tæller-værdi, så ens passwords aldrig ser ens ud i databasen.

## Opsætning af Projektet

### Database Migrations
For at oprette tabellerne i databasen, kør følgende kommandoer i `src/KeyVaults/KeyVaults.Infrastructure` (og tilsvarende mapper for andre services):

```powershell
$env:ASPNETCORE_ENVIRONMENT="Development"
dotnet ef database update --context SecurityKeyContext --project . --startup-project ../KeyVaults.Api.Service
```

### Kør Frontend
Kræver Flutter SDK installeret.

```bash
cd PasswordManager.Client
flutter pub get
flutter run
```
