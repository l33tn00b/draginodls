# draginodls
Setting Up Dragino D-LS GPS Tracker for TTN

Just some quick notes:

- make sure to set channel SF12!

<img width="557" height="653" alt="grafik" src="https://github.com/user-attachments/assets/d1730d59-e69d-4018-9699-91d1389c7926" />
 
- Sending AT Commands is a pain:
<img width="578" height="392" alt="grafik" src="https://github.com/user-attachments/assets/0f39742e-21f2-493e-9260-e92db0fe57db" />

Single char input doesn't work. You need to paste the entire command at once into the terminal window. This is the intended behaviour. As per doc: 
```
7.8  Why when using some serial consoles, only inputting the first string port console will return "error"?

Need to enter the entire command at once, not a single character.
User can open a command window or copy the entire command to the serial console.
```
