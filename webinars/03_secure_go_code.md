# Security Best Practices for Go Applications in the Cloud

## About

In this session we will consider the challenges in writing secure Go applications for the cloud

> Based on the 2019 talk [Going Secure with Go](https://www.youtube.com/watch?v=9e2gRtzemGo) by Natalie Pistunovich

## About the author/presenter

## Challenges

- How to secure the data we handle
- How to secure the code we write

## Solution

## Data

### Passwords

#### Generating Passwords

- Easy! That's what rand is for.

```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	rand.Seed(1)
	b := make([]byte, 4)
	rand.Read(b)
	fmt.Println(b)
}
```

```console
root@6178550486dd:/gen# go run main.go
[82 253 252 7]
root@6178550486dd:/gen# go run main.go
[82 253 252 7]
root@6178550486dd:/gen# go run main.go
[82 253 252 7]
root@6178550486dd:/gen#
```

> Oops! We forgot to seed properly.

- No problem - fixed!

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	rand.Seed(time.Now().Unix())
	b := make([]byte, 4)
	rand.Read(b)
	fmt.Println(b)
}
```

```console
root@6178550486dd:/gen# go run main.go
[48 8 59 67]
root@6178550486dd:/gen# go run main.go
[14 141 48 102]
root@6178550486dd:/gen# go run main.go
[227 197 229 124]
root@6178550486dd:/gen# go run main.go
[227 197 229 124]
root@6178550486dd:/gen# go run main.go
[227 197 229 124]
root@6178550486dd:/gen# go run main.go
[65 139 251 40]
root@6178550486dd:/gen#
```

> Not quite! If you're fast enough you get duplicate passwords!

- Use crypto/rand

```go
package main

import (
	"crypto/rand"
	"fmt"
)

func main() {
	b := make([]byte, 4)
	rand.Read(b)
	fmt.Println(b)
}
```

```console
root@6178550486dd:/gen# go run main.go
[8 144 182 26]
root@6178550486dd:/gen# go run main.go
[254 59 102 7]
root@6178550486dd:/gen# go run main.go
[189 105 28 145]
root@6178550486dd:/gen# go run main.go
[2 69 148 91]
root@6178550486dd:/gen#
```

> But what does it do?

```go
// Reader is a global, shared instance of a cryptographically
// secure random number generator.
//
// On Linux and FreeBSD, Reader uses getrandom(2) if available, /dev/urandom otherwise.
// On OpenBSD, Reader uses getentropy(2).
// On other Unix-like systems, Reader reads from /dev/urandom.
// On Windows systems, Reader uses the CryptGenRandom API.
// On Wasm, Reader uses the Web Crypto API.
var Reader io.Reader
```

##### Summary
    - Use [crypto/rand](https://pkg.go.dev/crypto/rand) for randomness
    - Ensure the underlying random number generator

#### Storing Passwords (secrets management)

- I'll just hard-code this secret in the code! -> show google search for DB_PASSWORD and strike trough on next slide
- Oops! I'll just quickly delete it from the git repo. -> not a good idea either
- I'll use a secrets manager (e.g. [kubernetes](https://v1-13.docs.kubernetes.io/docs/concepts/configuration/secret/))!
    - How are secretes stored in kubernetes?
    - Who has access to the secrets? Who can modify them?
    - How can I see who has accessed the secrets or modified them?
    - How are the secrets transferred to the application?
- env variables - https://kubernetes.io/docs/concepts/configuration/secret/
- process namespace sharing https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/#understanding-process-namespace-sharing

> You should be fine with kubernetes so long as you follow the best practices described in their [security best practices](https://kubernetes.io/docs/concepts/configuration/secret/#best-practices)

##### Summary
    - Consider the questions above for any secrets management tool you use.
    - Find and follow the established security best practices.
    - Use a dedicated secrets management tool e.g. [Hashicorp Vault](https://www.vaultproject.io/)

#### Storing User Passwords

- How about we just save in the DB?

```go
func saveUser(db *sql.DB, username, password string) error {
	query := fmt.Sprintf(
		"INSERT INTO users VALUES('%s', '%s')",
		username,
		password)
	_, err := db.Exec(query)
	return err
}
```

> Seems like a bad idea! When we select from the db we see:

```properties
Jim:abc
John:password
Leah:password
Phillip:8!7m!2iUNeNTdo!V
```

- Maybe we should encrypt it?

```go
func encrypt(password, passphrase string) string {
	block, _ := aes.NewCipher([]byte(createHash(passphrase)))
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		panic(err.Error())
	}
	nonce := make([]byte, gcm.NonceSize())
	if _, err = io.ReadFull(rand.Reader, nonce); err != nil {
		panic(err.Error())
	}
	return fmt.Sprintf("%X",
		gcm.Seal(nonce, nonce, []byte(password), nil))
}

func saveUser(db *sql.DB, username, password string) error {
	query := fmt.Sprintf(
		"INSERT INTO users VALUES('%s', '%s')",
		username,
		encrypt(password, passphrase))
	_, err := db.Exec(query)
	return err
}
```

> Great! Now we see only nonsense and we can always restore the users password!:

```properties
Jim:070F1C554385C39A6370D1BE22C021262545A4CA502F947F56A6E087A3ED78
John:49CABFE723F31759F3B01B901393EE3E60064EBA84BD6060D95EF1BBC84F0D14438F331B
Leah:0C27194BE30695AE6609020A70D983997DC5614F645E53C84063804FF6FF7C7EDE3F172C
Phillip:9C7BCBDBF528C0C745E33006529BD6D349FA660DFDAE37B41DA17646B26EA7DCEC1CF18EA16CEF318C5B2A91
```

> Still a bad idea since if a dump of the db is stolen together with the passphrase hackers get all the passwords.

- So encryption - not so good. Maybe crypto hash?

```go
func cryptohash(password string) string {
	return fmt.Sprintf("%X", sha256.Sum256([]byte(password)))
}

func saveUser(db *sql.DB, username, password string) error {
	query := fmt.Sprintf(
		"INSERT INTO users VALUES('%s', '%s')",
		username,
		cryptohash(password))
	_, err := db.Exec(query)
	return err
}
```

> Great! Now we see only nonsense and no one can guess the passwords!

```properties
Leah:5E884898DA28047151D0E56F8DC6292773603D0D6AABBDD62A11EF721D1542D8
Phillip:2380D3187175AD598B0DDBA3F7C760F02E79F6434A9D37906D09574D720E5546
Jim:BA7816BF8F01CFEA414140DE5DAE2223B00361A396177A9CB410FF61F20015AD
John:5E884898DA28047151D0E56F8DC6292773603D0D6AABBDD62A11EF721D1542D8
```

> But not really since it is pretty clear that John and Leah have the same password!
> Worse - password dictionaries can be hashed to easily check against a db of hashed passwords.

- Getting close now! Lets add some salt!

```go
func cryptohash(password string) string {
	bytes := make([]byte, 16, len(password)+16)
	rand.Read(bytes)
	bytes = append(bytes, []byte(password)...)
	hash := sha256.Sum256(bytes)
	return fmt.Sprintf("%X:%X", bytes[:16], hash[:])
}
```

> Great! Now we see only nonsense and all the passwords are different! Also we could have just let postgresql [handle it for us](https://www.postgresql.org/docs/current/pgcrypto.html) ... Oops.

```properties
Jim:09DA1C7D35AD46411982AB0C8055FE08:37F7307E1F994FF8B3E6A8A18643F45F26F0C5194159AE9F1C03B899892D42B3
John:9992C4118060A037EE367A31BCE73FDB:D1DB2024E210B5A12C039F456BA541E4122A2B1C93FDF3AD2406A16D7015ACEC
Leah:707F1969EC519F82063FFF372A5E6830:48C9D74EAFA6BDB8A47908234F0C09E81D4B3FBD38193BA26A55A10B3CC72DDE
Phillip:6E4E234194AEA7120C5AFF3F1D4280C6:920B71313157FFE53745707F7638223F8F3C5BE87992682873C6F6D66B6DA47D
```

> But it is still not enough! Hardware is so fast that hackers can calculate the salted hash for a password dictionary and compare it to a leaked db in reasonable time.

- The answer (for now) - use a hash stretching algorithm such as [PBKDF2](http://en.wikipedia.org/wiki/PBKDF2), [bcrypt](http://en.wikipedia.org/wiki/Bcrypt) or [scrypt](http://en.wikipedia.org/wiki/Scrypt).

```go
func cryptokey(password string) string {
	salt := make([]byte, 16)
	rand.Read(salt)
	key, _ := scrypt.Key([]byte(password), salt, 32768, 8, 1, 32)
	return fmt.Sprintf("%X:%d:%d:%d:%d:%X",
		salt, 32768, 8, 1, 32, key)
}
```

```properties
Jim:B4A6A21C3C5BBB2D128A4FA2640C6A78:32768:8:1:32:F90515B7F8C87E6E9D130FA7ACAFA1F105754079EC93520127E820101A11E6E8
John:36C9E1C9DC3B80A11852BF38C334C674:32768:8:1:32:D43058243F94E706D68019B0E9D51BA05E85F778AF8B3067D6EF2908AB8680BA
Leah:A9982641EA1E7556C51074AEE71919EF:32768:8:1:32:58BAA4089229CF296E0859158A6F987C378ABCA6FD15D25C03243A67E3A3DF0A
Phillip:638CE1F03392F86B61CBA77EFAA9A4BF:32768:8:1:32:2B0C411BAECF227DD5220EF5C5236D8ACADA163129A71A62E59B4045EE0ABFC9
```

##### Summary

- Use a strong random number generator to create a salt of 16 bytes or longer.
- Feed the salt and the password into a hash stretching algorithm.
- Store the the salt, cost parameters and the final hash in your password database.
- Adjust your cost parameters regularly to keep up with faster cracking tools.
- Don’t try to knit your own password storage algorithm.

##### References
- [How to Store Your Users' Passwords Safely](https://nakedsecurity.sophos.com/2013/11/20/serious-security-how-to-store-your-users-passwords-safely/) goes into more details and is worth reading.
- The [crypto/sha256](https://golang.org/pkg/crypto/sha256/) package
- The [crypto/hmac](https://golang.org/pkg/crypto/hmac/) package
- The [x/crypto/scrypt](https://godoc.org/golang.org/x/crypto/scrypt) package

### Personal Data

- Personal data can leak via logs!

> In regions where the EU General Data Protection Regulation (GDPR) privacy laws apply, the IP addresses and location data tracked in website server logs are also considered personal data.

- Use masking tools such as [sumo logic](https://help.sumologic.com/Manage/Collection/Processing-Rules/Mask-Rules), [splunk](https://docs.splunk.com/Documentation/Splunk/8.0.3/Data/Anonymizedata), [fluentd](https://www.fluentd.org/) to mask specific fields from the logs

## Code

### Input Validation

#### Unmarshaling

- Don't use XML as a data language if possible
- Be mindful of language specific tags when using YAML. Check the documentation for your parser.
- Check any dependencies you are using to implement RESTful endpoints with JSON. Some servers allow xml and json by default and xml has to be disabled explicitly.

> TODO more samples/code needed?

#### Validation

- Let's validate our user registration form

```go
type UserInfo struct {
	Username  string `json:"logon"`
	FirstName string `json:"first-name"`
	LastName  string `json:"last-name"`
	Email     string `json:"email"`
}

func (u UserInfo) IsValid() bool {
	if u.Username == "" || u.FirstName == "" || u.LastName == "" || u.Email == "" {
		return false
	}
	if len(u.Username) < 3 || len(u.Username) > 40 {
		return false
	}
	m, err := regexp.MatchString(emailRegex, u.Email)
	if err != nil {
		return false
	}
	return m
}
```

- Now let's do it in the idiomatic way

```go
import "github.com/go-playground/validator"

type UserInfo struct {
	Username  string `json:"logon" validate:"required,min=3,max=40"`
	FirstName string `json:"first-name" validate:"required"`
	LastName  string `json:"last-name" validate:"required"`
	Email     string `json:"email" validate:"required,email"`
}

func (u UserInfo) IsValid(validate validator.Validate) error {
	return validate.Struct(u)
}
```

##### Summary

- Use a library (such as [go-playground/validator](https://github.com/go-playground/validator)) for validating your structs after unmarshaling
- Create the validator once and pass it around as a dependency
- A domain entity can be part of several use-cases with different valid states. Don't be afraid to crete multiple structs and copy/paste the data around.
- You can add custom errors and even translators to the validator to output customer facing error messages directly.

###### References

- [XML_External_Entity_(XXE)_Processing](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing). Must read if you ever plan on using xml.
- The [go-playground/validator](https://github.com/go-playground/validator) project.

### SQL Injection

- Let's implement a products API

```go
func findProduct(db *sql.DB, key string) (ProductInfo, error) {
	query := fmt.Sprintf(
		"SELECT * FROM products WHERE product_key='%s'",
		key)
	rows, err := db.Query(query)
	// ... scan rows into product and handle err
	return product, nil
}
```

```go
func findProduct(db *sql.DB, key string) (ProductInfo, error) {
	query := "SELECT * FROM products WHERE product_key=$1"
	rows, err := db.Query(query, key)
	// ... scan rows into product and handle err
	return product, nil
}
```

- How about adding new products

```go
func addProducts(db *sql.DB, products []ProductInfo) error {
	var values []string
	for _, p := range products {
		values = append(values,
			fmt.Sprintf("('%s', '%s')", p.key, p.desc))
	}
	query := fmt.Sprintf(
		"INSERT INTO products VALUES %s",
		strings.Join(values, ","))
	_, err := db.Exec(query)
	if err != nil {
		return err
	}
	return nil
}
```

```go
func addProducts(db *sql.DB, products []ProductInfo) error {
	stmt, err := db.Prepare("INSERT INTO products VALUES($1, $2)")
	if err != nil {
		return err
	}
	defer stmt.Close()
	for _, p := range products {
		if _, err := stmt.Exec(p.key, p.desc); err != nil {
			return err
		}
	}
	return nil
}
```

##### Summary

- Never use the fmt package to construct sql queries
- Be mindful of the syntax and specifics of your chosen DB
- Read up on the sql package before implementing DB communication

##### References

- The [database/sql](https://golang.org/pkg/database/sql/) package
- [How Can I Prevent SQL Injection Attacks in Go](https://lmgtfy.com/?q=how+can+I+prevent+SQL+injection+attacks+in+Go)

### Static Code Analysis

- Benefits of automation
	- Safety net
	- Efficient onboarding

- [gosec](https://github.com/securego/gosec)

```console
root@6178550486dd:/sql# gosec ./...
[gosec] 2020/04/05 13:42:09 Including rules: default
[gosec] 2020/04/05 13:42:09 Excluding rules: default
[gosec] 2020/04/05 13:42:09 Import directory: /sql
[gosec] 2020/04/05 13:42:09 Checking package: main
[gosec] 2020/04/05 13:42:09 Checking file: /sql/main.go
Results:

[/sql/main.go:45-47] - G201 (CWE-89): SQL string formatting (Confidence: HIGH, Severity: MEDIUM)
  > fmt.Sprintf(
                "SELECT * FROM products WHERE product_key='%s'",
                key)

[/sql/main.go:88-90] - G201 (CWE-89): SQL string formatting (Confidence: HIGH, Severity: MEDIUM)
  > fmt.Sprintf(
                "INSERT INTO products VALUES %s",
                strings.Join(values, ","))

Summary:
   Files: 1
   Lines: 162
   Nosec: 0
   Issues: 2
```

- [depguard](https://github.com/OpenPeeDeeP/depguard)
	- In case you want to blacklist a certain library
	- Also depguard is part of ...
- [golangci-lint](https://github.com/golangci/golangci-lint)

## Closing words

## References
