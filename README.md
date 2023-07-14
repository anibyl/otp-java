# OTP-Java

[![Build & Test](https://github.com/BastiaanJansen/otp-java/actions/workflows/build.yml/badge.svg?branch=main)](https://github.com/BastiaanJansen/otp-java/actions/workflows/build.yml)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/91d3addee9e94a0cad9436601d4a4e1e)](https://www.codacy.com/gh/BastiaanJansen/OTP-Java/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=BastiaanJansen/OTP-Java&amp;utm_campaign=Badge_Grade)
![](https://img.shields.io/github/license/BastiaanJansen/OTP-Java)
![](https://img.shields.io/github/issues/BastiaanJansen/OTP-Java)

A small and easy-to-use one-time password generator for Java implementing [RFC 4226](https://tools.ietf.org/html/rfc4226) (HOTP) and [RFC 6238](https://tools.ietf.org/html/rfc6238) (TOTP).

## Table of Contents

* [Features](#features)
* [Installation](#installation)
* [Usage](#usage)
    * [HOTP (Counter-based one-time passwords)](#counter-based-one-time-passwords)
    * [TOTP (Time-based one-time passwords)](#time-based-one-time-passwords)
    * [Recovery codes](#recovery-codes)

## Features
The following features are supported:
1. Generation of secrets
2. Time-based one-time password (TOTP, RFC 6238) generation based on current time, specific time, OTPAuth URI and more for different HMAC algorithms.
3. HMAC-based one-time password (HOTP, RFC 4226) generation based on counter and OTPAuth URI.
4. Verification of one-time passwords
5. Generation of OTP Auth URI's

## Installation
### Maven
```xml
<dependency>
    <groupId>com.github.bastiaanjansen</groupId>
    <artifactId>otp-java</artifactId>
    <version>2.0.2</version>
</dependency>
```

### Gradle
```gradle
implementation 'com.github.bastiaanjansen:otp-java:2.0.2'
```

### Scala SBT
```scala
libraryDependencies += "com.github.bastiaanjansen" % "otp-java" % "2.0.2"
```

### Apache Ivy
```xml
<dependency org="com.github.bastiaanjansen" name="otp-java" rev="2.0.2" />
```

Or you can download the source from the [GitHub releases page](https://github.com/BastiaanJansen/OTP-Java/releases).

## Usage
### HOTP (Counter-based one-time passwords)
#### Initialization HOTP instance
To create a `HOTPGenerator` instance, use the `HOTPGenerator.Builder` class as follows:

```java
byte[] secret = "VV3KOX7UQJ4KYAKOHMZPPH3US4CJIMH6F3ZKNB5C2OOBQ6V2KIYHM27Q".getBytes();
HOTPGenerator hotp = new HOTPGenerator.Builder(secret).build();
```
The above builder creates a HOTP instance with default values for passwordLength = 6 and algorithm = SHA1. Use the builder to change these defaults:
```java
HOTPGenerator hotp = new HOTPGenerator.Builder(secret)
        .withPasswordLength(8)
        .withAlgorithm(HMACAlgorithm.SHA256)
        .build();
```

When you don't already have a secret, you can let the library generate it:
```java
// To generate a secret with 160 bits
byte[] secret = SecretGenerator.generate();

// To generate a secret with a custom amount of bits
byte[] secret = SecretGenerator.generate(512);
```

It is also possible to create a HOTP instance based on an OTPAuth URI. When algorithm or digits are not specified, the default values will be used.
```java
URI uri = new URI("otpauth://hotp/issuer?secret=ABCDEFGHIJKLMNOP&algorithm=SHA1&digits=6&counter=8237");
HOTPGenerator hotp = HOTPGenerator.fromURI(uri);
```

Get information about the generator:

```java
int passwordLength = hotp.getPasswordLength(); // 6
HMACAlgorithm algorithm = hotp.getAlgorithm(); // HMACAlgorithm.SHA1
```

#### Generation of HOTP code
After creating an instance of the HOTP class, a code can be generated by using the `generate()` method:
```java
try {
    int counter = 5;
    String code = hotp.generate(counter);
    
    // To verify a token:
    boolean isValid = hotp.verify(code, counter);
    
    // Or verify with a delay window
    boolean isValid = hotp.verify(code, counter, 2);
} catch (IllegalStateException e) {
    // Handle error
}
```

### TOTP (Time-based one-time passwords)
#### Initialization TOTP instance
TOTP can accept more paramaters: `passwordLength`, `period`, `algorithm` and `secret`. The default values are: passwordLength = 6, period = 30 and algorithm = SHA1.

```java
// Generate a secret (or use your own secret)
byte[] secret = SecretGenerator.generate();

TOTPGenerator totp = new TOTPGenerator.Builder(secret)
        .withHOTPGenerator(builder -> {
            builder.withPasswordLength(6);
            builder.withAlgorithm(HMACAlgorithm.SHA1); // SHA256 and SHA512 are also supported
        })
        .withPeriod(Duration.ofSeconds(30))
        .build();
```
Or create a `TOTP` instance from an OTPAuth URI:
```java
URI uri = new URI("otpauth://totp/issuer?secret=ABCDEFGHIJKLMNOP&algorithm=SHA1&digits=6&period=30");
TOTPGenerator totpGenerator = TOTPGenerator.fromURI(uri);
```

Get information about the generator:
```java
byte[] secret = totpGenerator.getSecret();
int passwordLength = totpGenerator.getPasswordLength(); // 6
HMACAlgorithm algorithm = totpGenerator.getAlgorithm(); // HMACAlgorithm.SHA1
Duration period = totpGenerator.getPeriod(); // Duration.ofSeconds(30)
```

#### Generation of TOTP code
After creating an instance of the TOTP class, a code can be generated by using the `now()` method, similarly with the HOTP class:
```java
try {
    String code = totpGenerator.now();
     
    // To verify a token:
    boolean isValid = totpGenerator.verify(code);
} catch (IllegalStateException e) {
    // Handle error
}
```
The above code will generate a time-based one-time password based on the current time. The API supports, besides the current time, the creation of codes based on `timeSince1970` in seconds, `Date`, and `Instant`:

```java
try {
    // Based on current time
    totpGenerator.now();
    
    // Based on specific date
    totpGenerator.at(new Date());
    
    // Based on specific local date
    totpGenerator.at(LocalDate.of(2023, 3, 2));
    
    // Based on seconds past 1970
    totpGenerator.at(9238346823);
    
    // Based on an instant
    totpGenerator.at(Instant.now());
} catch (IllegalStateException e) {
    // Handle error
}
```

### Generation of OTPAuth URI's
To easily generate a OTPAuth URI for easy on-boarding, use the `getURI()` method for both `HOTP` and `TOTP`. Example for `TOTP`:
```java
TOTPGenerator totpGenerator = new TOTPGenerator.Builder(secret).build();

URI uri = totpGenerator.getURI("issuer", "account"); // otpauth://totp/issuer:account?period=30&digits=6&secret=SECRET&algorithm=SHA1

```

## Recovery Codes
Often, services provide "backup codes" or "recovery codes" which can be used when the user cannot access the 2FA device anymore. Often because 2FA device is a mobile phone, which can be lost or stolen. 

Because recovery code generation is not part of the specifications of OTP, it is not possible to generate recovery codes with this library and should be implemented seperately.

## Licence
OTP-Java is available under the MIT License. See the LICENCE for more info.

[![Stargazers repo roster for @BastiaanJansen/otp-java](https://reporoster.com/stars/BastiaanJansen/otp-java)](https://github.com/BastiaanJansen/otp-java/stargazers)
