---
title: "Encryption for .NET Developers Part I - Symmetric encryption"
date: 2019-04-27T17:31:53-04:00
draft: true
---

This is a series of posts about encryption for .NET developers.

## In this series

* [Intro]({{< ref "encryption-intro.md" >}})
* [Encryption for .NET Developers Part I - Symmetric encryption (this post)]({{< ref "encryption-i.md" >}})

## Symmetric Encryption

Symmetric encryption involves just a single "key". This key is shared between all parties that need to encrypt or decrypt information. For that reason, it can be considered less secure in an environment where multiple parties need to transmit information.

For this reason, it's usually used when the party that needs to encrypt is also the party that is decrypting. These scenarios include encrypting information to be transmitted to another party but not to be read by that party. For example, in some [OAuth](https://oauth.net/) flows, a parameter called `state` is submitted with the request. The value of this parameter is returned untouched back to the requestor. The requestor usually puts information in the state to track the request, such as a session ID or internal information. In this scenario, the requestor encrypts the data with a symmetric key and "stringifies" that result with Base64 encoding and places that in the `state` paramter.

### Keys

### Initialization Vector

## AES

[AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard), or **Advanced Encryption Standard** is one algorithm that performs this symmetric encryption.

Below is an example of AES encryption in C#:

{{< highlight csharp >}}
using System;
using System.IO;
using System.Security.Cryptography;

//...

public static void Encrypt()
{
    var plainText = "Hello, World!";

    using (AesCryptoServiceProvider aes = new AesCryptoServiceProvider())
    {

        ICryptoTransform encryptor = aes.CreateEncryptor(aes.Key, aes.IV);

        using (var ms = new MemoryStream())
        using (var cryptoStream = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
        {
            using (var sw = new StreamWriter(cryptoStream))
            {
                sw.Write(plainText);
            }

            var encrypted = ms.ToArray();
            Console.WriteLine(Convert.ToBase64String(encrypted));
        }
    }
}
{{< /highlight >}}

This produces an output like `UAxdT2aKiHCYRlUm662Xtw==`

One thing to note in this example is that the key is regenerated every time. When a new `AesCryptoServiceProvider` is constructed, a new **Key** and **Initilaization Vector** is created

{{< highlight csharp >}}
using (AesCryptoServiceProvider aes = new AesCryptoServiceProvider())
{{< /highlight >}}