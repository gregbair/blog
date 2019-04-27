---
title: "Encryption for .NET Developers Part I - Asymmetric encryption"
date: 2019-04-27T17:31:53-04:00
draft: true
---

This is a series of posts about encryption for .NET developers.

## In this series

* [Intro]({{< ref "encryption-intro.md" >}})
* [Encryption for .NET Developers Part I - Asymmetric encryption (this post)]({{< ref "encryption-i.md" >}})

## Asymmetric Encryption

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

This is where we are