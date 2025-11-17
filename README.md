# Werkzeug-PBKDF2-Hash-Converter

A simple python script for converting Werkzeug PBKDF2 hashes to their PBKDF2-HMAC-SHA256 Hachcat format and vice versa

This is a simple python script for converting pbkdf2_sha256 hashes into Hashcat compatible hashes. You can also work backwards from Hashcat compatible hashes into pbkdf2 hashes. 

The script also contains a batch function that allows for the conversion of multiple hashes from a single input file.

# Usage:  

                           python3 pbkdf2-hashconv.py

Follow prompts and it's that simple!

NOTE: After you place your converted hashcat compatible hash in a file (ie hash.txt), it can be cracked with the following command:

                   hashcat -m 10900 -a 0 hash.txt /path/to/wordlist.txt


<img width="958" height="763" alt="pbkdf2-hashconv-sample" src="https://github.com/user-attachments/assets/22d0f434-729f-4f5b-a2a1-7d97d8450a2d" />
