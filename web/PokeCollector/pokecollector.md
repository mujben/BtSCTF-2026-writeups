# Description
Opening card packs has become painfully expensive, so I wrote an app to open Pokemon packs. Mewtwo is mine though.

# Reconnaissance
## Web page
The application serves as web based Pokemon card opening simulator. Core functionalities consist of opening packs and obtaining random Pokemon cards for a user profile. The presence of registration and login forms suggests potential for a SQLi.
<img width="1599" height="747" alt="image" src="https://github.com/user-attachments/assets/6dbd2027-952f-47eb-8096-7e10109ebce9" />

Analysis of registration form returned no sign that default account like `admin` is present, as attempt to create such accounts resulted in success.
<img width="1601" height="748" alt="2 od początku" src="https://github.com/user-attachments/assets/1bc07ca8-cb48-4417-8160-d3cb4b123a8a" />

## Directory fuzzing
A directory brute force scan was performed using feroxbuster:
```
feroxbuster -u 'https://pokecollector[...].chall.bts.wh.edu.pl' \
 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -k -o scan.txt -t 100
```
The scan identified several open endpoints. The application uses `FastAPI` framework (Python). Due to FastAPI's modern SQL query handling it is improbable that the exercise consists of abusing SQLi schemes.
<img width="1599" height="865" alt="3 od początku" src="https://github.com/user-attachments/assets/8eb07b85-c6fb-47fd-8960-80dffdf05714" />

## Docs page
Accessing the `/docs` endpoint, revealed the OpenAPI specification and several interactive API routes.
<img width="1601" height="820" alt="4 od początku" src="https://github.com/user-attachments/assets/b933b476-2289-4f48-8405-3e0fe28c39bd" />
The most important one seems to be `/api/collection/add`, which accepts a JSON request body:
```
{
  "pokemon_id": 0,
  "pokemon_name": "string"
}
```
# Exploitation
## Request manipulation
Execution of the exploit required an authenticated user session. An initial attempt involved capturing the POST request after clicking on `OPEN PACKS` and later `OPEN NEW PACK` via BurpSuite.
By modifying request body to:
```
{
  "pokemon_id": 150,
  "pokemon_name": "Mewtwo"
}
```
a JWT was returned in response. Direct modification of the request through a proxy proved ineffective, necessitating a deeper analysis of the client-side logic.

## Client side JavaScript
The application source code includes a JavaScript file named `app.js`. It is responsible for handling Pokemon generation and adding them to user's account. 
<img width="998" height="502" alt="image" src="https://github.com/user-attachments/assets/2ced5242-9956-462d-b681-8c2f006f4607" />
An attempt to call a function responsible for adding a Pokemon to user's collection was made by typing below input to a browser console.
```
saveToCollection(150, "Mewtwo");
```
It resulted in immediate addition of desired Pokemon to user's collection, bypassing the randomization logic.
<img width="813" height="733" alt="pokecollector_btsctf" src="https://github.com/user-attachments/assets/15c9750c-8309-48e1-a2a2-002db1bc61a2" />

## Stateless pokemon storage
Further investigation of the server's responses revealed the use of JSON Web Tokens (JWT) for session management. Decrypting the token showed that the application employs a stateless storage model. The entire user collection is encoded directly within the JWT payload.
Each time a Pokemon is added, the server re-signs the token with the updated collection data provided by the client, without verifying the legitimacy of the Pokemon acquisition process.
<img width="1600" height="811" alt="2 od końca" src="https://github.com/user-attachments/assets/09e0e9b8-ea6a-4179-9acb-4f838ff652f0" />
<img width="1326" height="654" alt="1 od końca" src="https://github.com/user-attachments/assets/af966a22-f0a9-4467-aa7a-26530e8c3c47" />
# Conclusion
The vulnerability discovered is a classic "Insecure Business Logic" combined with "Trusting Client Input". 
The server-side validation was missing, allowing users to claim any Pokemon by ID. 
Additionally, storing the entire state in a JWT that is re-signed based on user-controlled input confirmed the stateless and insecure nature of the collection management.
