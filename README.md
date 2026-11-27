
# SL Departure Display – AWS IoT

En IoT-baserad lösning för visning av aktuella avgångstider inom Stockholms kollektivtrafik. Projektet kombinerar inbyggda system, molntjänster och ett webbaserat gränssnitt för fjärrkonfiguration av IoT-enheter.

## Projektöversikt

Projektets baktanke och syfte,
Vi vet alla hur stressigt det kan vara på morgonen och många är också bekanta med att SL:s avgångar inte alltid är i tid. När man gör sig redo för att bege sig hemifrån finns det inte alltid tid att kontrollera aktuella avgångstider.
Ur detta uppstod idén till projektet – att ha en fysisk display som i princip i realtid uppdaterar och visar information om valda avgångar, utan att användaren aktivt behöver kontrollera sin telefon.
Den fungerar på så sätt: 

På Web interface registrerar man sin enhet/display.

Väljer vilken hållplats, färdmedel, linje, riktning och uppdatering intervall. 

Då uppdaterar displayen den valda linjen på sin display och man kan även se avgången på websidan, samt ändra vilken avgång man vill följa.

- **Krav ställning:** Kunna se vald realtids avgång från SL, utan att behöva ta tiden att ta telefonen och söka på vald avgång.


## Funktionalitet - Användar guide

Steg 1.0
Skapa ett konto / loggar in:

<img width="500" alt="Skärmbild 2026-11-26 163433" src="https://github.com/user-attachments/assets/420e179a-78a9-4765-80ee-13a5a625fb7f" />

Steg 1.1
Första gången du startar enheten visas enhetens mac adress :

<img width="500" alt="f19a131f-0750-4d95-b68f-b97aace1b20a" src="https://github.com/user-attachments/assets/f66ff8df-9ba0-48de-8ad5-73b91cb05e99" />

Steg 2.0
Registerna din enhet genom att ange mac adressen och ge den ett namn, och en länk till discord-webhook (Om man vill få meddelande om enheten kopplas ifrån)  :

<img width="500" alt="Skärmbild 2026-11-26 163751" src="https://github.com/user-attachments/assets/2df2bf2e-2260-4e1c-842f-aba05d19e58f" />

Steg 2.1
Då ansluter enheten till molntjänst(erna), och ber dig ange vilken hållplats, färdmedel, linje, riktning och vliken uppdateringsintervall du vill ha:

<img width="500" alt="8b431e1d-dc6b-4276-8b9f-a34998e6be62" src="https://github.com/user-attachments/assets/b514a7ca-a71d-453a-abb2-31b2cd63c2a8" />

Steg 3.0
Ange hållplats, färdmedel, linje, riktning och uppdatering intervall :

<img width="500" alt="Skärmbild 2026-11-26 163909" src="https://github.com/user-attachments/assets/3a61fd8a-4ebb-4e37-8be6-aef3194e7c5a" />

Steg 3.1
Då sparas den valda avgången på enheten och den uppdaterar efter din valda intervall:

<img width="500" alt="8e8ec9e5-ca45-4711-bba4-21ad75105eef" src="https://github.com/user-attachments/assets/f9b4214e-d7bb-47d0-902a-5fa2880a203a" />
<img width="500" alt="Skärmbild 2026-11-27 100414" src="https://github.com/user-attachments/assets/92b353ff-fea0-4106-9db9-d2ffd05cc236" />

Felsökning och återställning :
Skulle enheten av nån anledning strula och eller inte kunna ansluta ska du välja "Ta bort vald enhet" -> Ok.   
Samt trycka på "Boot" knappen på enehten

<img width="500" alt="Skärmbild 2026-11-26 171116" src="https://github.com/user-attachments/assets/bbc1abdb-4742-421c-b630-94a59b59051a" />

<img width="500" alt="ea90337b-930b-4b73-8313-9048ed69979a" src="https://github.com/user-attachments/assets/6f21af78-f010-49a7-a0b0-93f0e671ab7f" />

Du får även då meddelandet på discord att din enhet har kopplats ifrån

<img width="500" alt="Skärmbild 2026-11-27 100442" src="https://github.com/user-attachments/assets/711d95d0-8a2c-40ca-af48-d63df71f7000" />



## Systemarkitektur, Flödesschema / Diagram,



<img width="901" height="1542" alt="flödesdiagram drawio (1)" src="https://github.com/user-attachments/assets/be46f976-09f3-4b47-8ff2-427823ffe132" />

Flödesschemat / diagrammet är uppdelat i två delar, 1 Hur första anslutningen funkar, 2 hur själva flödet går när enheten är uppkopplad.
Diagrammet ska läsas uppifrån och ner / från vänster till höger. 

### Anslutning och enhetsprovisionering

Vid första uppstart ansluter enheten till Wi-Fi (hårdkodat). Därefter kontrollerar enheten om den har ett certifikat och en privat nyckel sparade i sitt flashminne (NVS).

Om den inte har det visas enhetens MAC-adress på LCD-displayen. I bakgrunden gör enheten en HTTPS POST-request med sin MAC-adress till API Gateway var tionde sekund.

Detta triggar Lambda-funktionen `Device_Provisioning`, som i sin tur tar emot MAC-adressen och kontrollerar om enheten finns registrerad i DynamoDB.

- Om enheten finns registrerad skapas ett individuellt certifikat och en privat nyckel via AWS IoT Core. Dessa returneras i HTTPS-svaret och sparas i enhetens NVS.
- Om enheten inte finns registrerad fortsätter denna loop var tionde sekund.

Parallellt med detta ska användaren registrera den MAC-adress som visas på displayen i webbgränssnittet (Amplify). När användaren har lagt till enheten sparas den i DynamoDB med den inloggade användaren som ägare (ownership).

Lambda-funktionen `Device_Provisioning` kan då skapa och returnera certifikatet och den privata nyckeln.

Enheten kan därefter ansluta till AWS IoT Core via MQTT med mutual TLS (mTLS).

AWS:s CA-certifikat ligger fortfarande hårdkodat på enheten och används för att verifiera serverns certifikat.

### Dataflöde

Vid val av station finns alla stationer sparade i en separat DynamoDB-tabell. Denna uppdateras via en Lambda-funktion som hämtar alla stationer från SL/Trafiklabs API. Funktionen triggas just nu manuellt.

Stationerna hämtas via Amplify/AppSync och filtreras i React. Efter två inskrivna tecken visas matchande stationer via autocomplete i webbgränssnittet.

När vi har valt station skickar AppSync vald stations ID till Lambda-funktionen `SL_Station_Line_Service`.

Lambda-funktionen gör då en HTTPS GET-request till SL/Trafiklabs API och hämtar tillgängliga linjer, färdmedel och riktningar för den valda stationen.

Resultatet returneras sedan via AppSync till webbapplikationen och sparas tillfälligt i Reacts state. Användaren kan då välja färdmedel, linje och riktning utan att vi behöver göra ett nytt anrop till Lambda eller SL/Trafiklabs API för varje val.

När användaren väljer hållplats, färdmedel, linje, riktning och uppdateringsintervall och klickar på **Spara till Device Shadow**, triggar webbapplikationen via AppSync en Lambda-funktion som uppdaterar enhetens Device Shadow i AWS IoT Core.

Konfigurationen sparas i enhetens desired state. IoT Core skickar förändringarna till enheten via följande MQTT-topic:

`$aws/things/{deviceId}/shadow/update/delta`

Enheten tar emot och sparar den valda konfigurationen. Vid uppstart kan enheten även hämta sin befintliga Device Shadow via:

`$aws/things/{deviceId}/shadow/get`

Därefter skickar enheten en förfrågan om avgångsinformation till IoT Core via MQTT-topicen:

`{deviceId}/getDeparture`

Via IoT Rule `esp_get_departure` och message routing skickas denna data vidare till Lambda-funktionen som hanterar SL:s API.

Lambda-funktionen bygger själva URL:en för den valda hållplatsen, linjen, färdmedlet och riktningen och gör en HTTPS GET-request till SL/Trafiklabs API.

Lambda-funktionen plockar sedan ut den information vi är intresserade av, eftersom API-svaret innehåller mycket metadata:

- Linje
- Avgångstid
- Tid kvar till avgång
- Riktning

Lambda-funktionen publicerar den bearbetade datan till enheten via MQTT-topicen:

`{deviceId}/receiveDeparture`

Samma data publiceras även via AWS AppSync Events.

På så sätt visas samma avgångsinformation både på LCD-displayen och i webbapplikationen.

Det görs även en kontroll av om avgången är avvikande. Om en avvikelse identifieras sparas information om avgången i en Amazon S3-bucket.

För att undvika dubbletter används avgångens identifierare från SL/Trafiklabs API. Om avgången redan finns sparad ska den inte sparas på nytt.

### Frånkoppling och notifikationer

Om enheten skulle tappa anslutningen publicerar AWS IoT Core en frånkopplingshändelse på följande MQTT-topic:

`$aws/events/presence/disconnected/{clientId}`

En IoT Rule lyssnar på:

`$aws/events/presence/disconnected/#`

Denna regel triggar Lambda-funktionen `Forward_disconnet_notification`, som tar emot information om frånkopplingen, bland annat enhetens ID, tidpunkt och anledning till frånkopplingen.

Lambda-funktionen hämtar enhetens registrerade Discord Webhook från DynamoDB och skickar en notifikation till användarens Discord, förutsatt att användaren har konfigurerat en sådan.




## Teknologier och verktyg

| Område | Teknologier |
|---|---|
| Hårdvara | ESP32, LCD1602 |
| Firmware | C/C++, Arduino Ide|
| Molntjänster | AWS IoT Core, Lambda, DynamoDB, Amplify, AppSync, API Gateway, S3 |
| Autentisering | Amazon Cognito, X.509-certifikat |
| Loggning och övervakning | Amazon CloudWatch |
| Webbapplikation | React, TypeScript, Vite |
| Kommunikation | MQTT, HTTPS |
| Dataformat | JSON |
| Externa integrationer | SL Transport API (Trafiklab), Discord Webhooks |


## Säkerhet och skalbarhet

- **Automatiserad enhetsprovisionering:** Nya IoT-enheter registreras och tilldelas individuella certifikat och privata nycklar automatiskt.
- **Säker kommunikation:** Kommunikation mellan ESP32 och AWS IoT Core sker via MQTT över mutual TLS (mTLS) med X.509-certifikat. 
- **Användarautentisering:** Amazon Cognito används för registrering, inloggning och autentisering av användare.
- **Behörighetshantering:** IoT Policies och kontroll av enheternas ägarskap används för att begränsa åtkomsten till systemets resurser.
- **Serverlös arkitektur:** Genom AWS Lambda, DynamoDB och övriga hanterade molntjänster kan systemet skalas efter antalet användare, enheter och API-anrop, utan att behöva administrera egna servrar.



## Framtida utveckling

- Schemalägga uppdateringen av hållplatser i DynamoDB, som idag sker manuellt.
- Försträcka enhets validering med mer än bara Mac-Adress som validering / registrering
- Ha en möjlighet att kunna justera Esp32 Wifi ssid - password via Bluetooth eller Wifi AP
- Mer utvecklat Web ui, där man kan se avvikelserna från s3 bucket
- Kunna sätta enheten i sleep mode och bara köra / visa avgångar på bestämd tids intervall tex bara mellan 07.00 - 08.00
