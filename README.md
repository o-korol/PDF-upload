# CHATBOT ASSIGNMENT COMPLETION TOKENS
#### Video Demo:  https://youtu.be/6wAa-mWPEvM
#### Description:

## Table of Contents
1. [Project goal](#project-goal)
2. [Design considerations](#design-considerations)
3. [Components](#components)
4. [Installation](#installation)
5. [Usage](#usage)
6. [Collaboration with GPT-4](#collaboration-with-gpt4)

## Project goal
AI-powered chatbots, such as tutor bots or practice bots, can be useful educational tools.  Integration with learning management systems (LMS) is likely to lag behind chatbot development, so it would be useful to develop a tool that allows instructors to confirm that a student interacted with a particular chatbot.  The goal of this project is to develop a token that can be passed between a chatbot and LMS to confirm the interaction.

The following discussion assumes that the token will be issued by a practice bot.  A practice bot is a conceptual variation on tutor bot, one that comes preloaded with a set of problems related to a particular topic.  The bot is prompted to work through those problems with a student.  The student will be asked to "make their thinking visible" and the bot will be asked to provide feedback.  At the end of the session, the bot will be asked to evaluate the level of competence the student reached (e.g., "competent" or "developing").

### Token requirements

1. The token should contain information about the level of competence the student achieved (thereafter referred to as "grade").
2. The grade information in the token should *not* be transparent to the student, to avoid the temptation of fudging the grade.
3. The token should be unique, so that a student cannot "recycle" a token from a past assignment or share a token with a friend.

## Design considerations

### Creating unique tokens using timestamp

Uniqueness can be achieved by including a timestamp in the token.  datetime() function generates a timestamp in year-month-day hour:minutes:seconds:microseconds format.  Using microseconds alone could be sufficient, since microseconds include 6 significant digits, providing 1 million different combinations.  However, it may be useful to include year-month-date information as well, since a stale token may indicate recycling.

### Timestamp format

Adoption of AES influenced the choice of the token's timestamp format.  AES encrypts in blocks of 16 bytes, meaning that a plaintext of 16 characters will be encoded as 16 bytes, but a plaintext of 17 characters will be encoded as 32 bytes. Thus, an effort was made to reduce the timestamp under 16 characters (to leave room for grade). Year was expressed in 2-digit format (23 rather than 2023) and all extraneous characters (like - and :) were removed; only 2 decimal places were kept.

### Logging tokens
To ensure that tokens are not recycled or shared, newly-submitted tokens need to be checked against previously submitted tokens.  This project chose to store tokens in a database, rather than in a dictionary written to a .csv file.  The database was indexed, to speed up search.

Python dictionaries utilize hash tables to store data, making them similarly fast to search, but opening a .csv file, reading it, and loading it into a dictionary is likely to take longer than accessing a database.

(Users have an option of downloading the log to a .csv file at the end of the session.)

What is logged:
- encrypted token,
- decrypted token,
- token date,
- age,
- grade,
- whether the token is a duplicate (if it is, what are the IDs of tokens it is a duplicate of),
- whether it has been duplicated.

In addition, a problem flag is logged. (It tags tokens that are duplicates, or have been duplicated, or cannot be decrypted.)


### Ensuring non-transparency via encryption

Non-transparency can be achieved by encryption.  A simple cipher easily decodable by the instructor, such as Caesar's cipher, would be convenient, but it can be easily cracked by students, who are likely to complete multiple chatbot assignments during semester and thus accumulate a collection of ciphertext and plaintext tokens.

AES encryption was chosen for this project for several reasons:
1. Like hash, AES output is sensitive to small changes in input.  This is important because plaintext tokens contain a timestamp and thus are quite similar to each other (e.g., C**230101**93812 vs C**230101**93983).
2. It is more secure than other algorithms, such as RC4, Salsa20, or ChaCha20 stream ciphers.  In particular, stream ciphers are vulnerable if the same key is used more than once.

### AES choices

AES offers several **modes**, including Electronic Code book (ECB) and Cipher Block Chaining (CBC). CBC offers greater security, but ECB offers a shorter ciphertext, since the ciphertext does not have the initialization vector (IV) prepended to it.  This tilted the scale in favor of ECB mode.

AES offers several different **key lengths** (16, 24, and 32 bytes).  Longer keys provide more security, but require longer encryption/decryption times.  In this case, individual tokens are short (16 bytes) and decryption batches are expected to be small (less than 100 tokens), so encryption/decryption time is not expected to be a limiting factor.  The longest key (32 bytes) was chosen for this project.

AES requires **secure storage of the key**.  The key can be stored in an environmental variable, a secure file, a secure database, or a key management service like AWS KMS or Google Cloud KMS.  Since this is not a final product but a demo, this project chose to store the key in an environmental variable for the sake of simplicity.  One of the downsides of this approach is that an environmental variable persists only during a session.

### Transformation of encrypted token from binary to base64

AES generates a binary literal, which is cryptic-looking and relatively long.  To make it shorter and more human-friendly-looking, binary output was transformed into base64 string.

Transformation from binary to a string is necessary, since students are expected to copy-paste the token into LMS.  Binary does not copy-paste well.

## Components
### tokens.py file
Contains the entire Python code for token generation, encryption, decryption, and logging.  Since it is a demo, the grade is generated randomly.  The encryption key is stored in an environmental variable, so it does not persist between sessions.

### tokens.db file
Contains a database that logs tokens.

Database schema:

CREATE TABLE sqlite_sequence(name,seq);
CREATE TABLE tokens (id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, ciphertext TEXT NOT NULL, token TEXT, date NUMERIC, age NUMERIC, grade TEXT, duplicate INTEGER, duplicated INTEGER, problem INTEGER NOT NULL, duplicate_of TEXT);

To speed up search, the database was indexed on tokens column, which contains decrypted tokens:
CREATE INDEX idx_token ON tokens (token);

### app.py, helpers.py, templates folder, static folder
Contain files to demo the code on the web.  Since this is a demo, the grade is generated randomly.
Keep in mind that the encryption key is stored in an environmental variable, so it will be reset every time the server restarts.

### full_log and problem_log files
.csv files that contain information downloaded from the database.
(The code is commented out, so that new files are not generated every time the code is run.)

## Installation
This project requires Python 3 and PyCryptodome library.

You can check your Python version with:
```bash
python --version
```

You should see 'Python 3.x.x'.

You can install PyCryptodome library, used for AES encryption, via this command in your terminal:
```bash
pip install pycryptodome
```

THE git PART NOT OPERATIONAL YET: the project is not available on GitHub yet

After installing the necessary library, you can clone the project and run it locally:

```bash
git clone https://github.com/o-korol/project.git
cd yourproject
python app.py
```

Download the project files.  Extract the files.  Navigate to the directory storing the extracted files and run:
```bash
python tokens.py
```

Alternatively, run the web demo (app.py) using
```bash
flask run
```
Note: The encryption key is stored in an environmental variable, so it will be reset every time the server restarts.

## Usage

This section will guide you through the basic usage of the application. The application runs as a Flask server.

### Generate a token

1. Navigate to the `Get Token (students)` page of the application via the navbar or click on `Get token` button on the homepage.
2. A randomly generated grade and a token will be displayed in the text field.  (The grade is randomly generated for demo purposes.) This token is encrypted and unique for each generation.
3.  Copy the token.

### Decrypt a token

1. Navigate to the `Check Tokens (instructors)` page of the application via the navbar.
2. Paste the encrypted token that you copied in the text field and click on the `Submit token` button.
3. The decrypted token will be displayed if the decryption is successful, accompanied by information about the token generation date, age (in days), grade, and duplicate status.
4. If the decryption is not successful, all fields other than ID and Ciphertext will be showing NULL.
5. Problematic tokens--tokens that cannot be decrypted, are duplicates, or have been duplicated--will be highlighted in red.  (To generate a sample problematic token, submit the same encrypted token more than once.)

### Log Tokens

1. Navigate to the `Log (instructors)` page of the application.
2. Here you can see a log of all the generated tokens, their decrypted form, timestamps, grade, etc.  Problematic tokens are highlighted in red.

## Collaboration with GPT4

This project was completed in collaboration with GPT-4.  I am aware that the course syllabus specified that no AI tools could be used, but since CS50.ai was made available to students, I felt justified in using GPT-4 with similar limitations.  I chose to use GPT-4 instead of CS50.ai in order to retain chat history.

Here's the relevant part of GPT-4 "personality prompt":

"When I have a code-related question, along the lines of "help me figure out how to do this" or "what is wrong with this code," please do not give the full correct code.  Instead, help me figure things out and help me understand.  It is OK to give me snippets of the code, just not the full code."

Here's the link to the actual chat: https://chat.openai.com/share/c1516feb-3a3e-4c1e-b695-ad181eba9404
