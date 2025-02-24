## Description
No description
## Hints0
- Something looks out of place with checkout
- It's always nice to get free stuff
## Flag0
This is the website of the challenge:

![Screenshot 2024-07-27 225528](https://github.com/user-attachments/assets/57f1c379-9db8-4a10-97b7-290ec05520cf)
Due to the hint we could try to change the price to 0, we got the first flag.

![Screenshot 2024-07-28 170427](https://github.com/user-attachments/assets/b26c0019-6ca8-4717-8b2d-a3cea0733540)
## Hints1
- There must be a way to administer the app
- Tools may help you find the entrypoint
- Tools are also great for finding credentials
## Flag1
To find a way to administer the app, we could brute force the other paths. I used dirb command with a list of common names: [here](https://github.com/danielmiessler/SecLists/tree/master/Usernames/Names).

![Screenshot 2024-07-28 181004](https://github.com/user-attachments/assets/4e52a3fe-d2fc-484d-83b4-1af95d3e23e1)

After some time, I investigated the `/login` path.

I tried some injections here, but they didn't work, so we had to brute-force again.

I used Turbo Intruder extension of BurpSuite, first i sent bruteforce username with [this](https://github.com/danielmiessler/SecLists/tree/master/Usernames/Names) common name list. If the username exist, it doesn't return `Invalid username`, so I filtered it out by the following code:

![Screenshot 2024-07-28 175956](https://github.com/user-attachments/assets/a41bd980-bde9-4335-a7f0-fb2e5cc8da8a)

Then I repeated the process to find the password using [this](https://github.com/danielmiessler/SecLists/blob/master/Passwords/Common-Credentials/10-million-password-list-top-10000.txt) word list, filtering by  `Invalid password`.

![Screenshot 2024-07-28 180603](https://github.com/user-attachments/assets/4c72e643-f6d8-4de8-b392-839767616a5e)

After obtaining the username and password, I logged into the website and got the second flag.

## Hints2
- Always test every input
- Bugs don't always appear in a place where the data is entered
## Flag2

Now that we administered the website and had more places to exploit, I tried some XSS on the `/edit` page and got the final flag when went back to the `/cart` page.

![Screenshot 2024-07-28 181315](https://github.com/user-attachments/assets/923cf927-78ae-4584-8103-1717960c0bb9)

I want to give my thanks to [@ujjwalkhadkaofficial](https://www.youtube.com/watch?v=CWTFJS9yNJA&t=118s&ab_channel=ujjwalkhadkaofficial) for his write-ups, i had no idea what to do before reading it.
