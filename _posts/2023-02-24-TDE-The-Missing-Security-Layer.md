---
title: "Transparent Data Encryption, the missing security layer for enterprise-grade Postgres" 
date: 2023-02-24
---

# Introduction 
Today, data is regarded as the most important asset an organization has. As businesses and government agencies realize the importance of data ownership and portability, there’s been a massive increase in enterprise adoption of open source databases such as PostgreSQL. However, maintaining data security and integrity in your database can be a complex task. 

Securing data at rest is largely a solved problem, with various options that provide coverage at the disk or within databases at the transaction level. These options provide flexibility but are meant to address different types of threats an organization may face. As data security concerns rise amongst organizations that have accelerated their cloud journey, data encryption needs to be a best practice. It can help safeguard confidential data and other cloud data assets from accidental exposure and unauthorized access, and it allows organizations to create a security architecture that mitigates numerous threats that could otherwise contribute to a security breach. 

Database security is just one of many security layers an organization needs to consider. But it’s the deepest one – without which organizations are likely to find themselves at greater risk in a challenging cyberspace. 

# The importance of securing your data

Whether or not you’re familiar with the field of cryptography and its effect on world history, you rely on cryptography daily to safeguard communications with your banking, internet search queries and online healthcare providers. When making an online purchase, you rightfully assume the party you are interacting with has gone to great lengths to ensure your connection is secured and your information can't be read by a third party in transit. But, when you log back onto the site, chances are the merchant has stored your credit card information for future use. And when you log into your health care portal with your personally identifiable information, details about yourself are visible on your device. 

Fortunately, internet standards and programming frameworks have made communication between devices easy to secure. HTTP Secure (HTTPS) is the default protocol nowadays and chances are you pay little attention to it. However, the place where your data persists, like the credit card stored for your convenience with an online merchant, or your healthcare information, often goes unprotected.

If you have a basic understanding of database technology, you might assume that someone has gone through the exercise of locking down the database and establishing rules such as who can connect, where a database connection be made from, database table and column permissions, restricted views and so on. But these safeguards are database-level protections, which only address one layer of an overall security strategy.

The other layer of your database security strategy must protect against the inevitable event that the malicious actor has access to the operating system where your database runs. This breach can occur if a skilled hacker or nation-state has exploited a vulnerability in one of the many defenses. If a malicious actor has access to the operating system, they don't need to go through a database connection.


# Protecting your database backups

Once hackers or insiders have access to the operating system, they have plenty of tools to bypass database controls and read data from the binary files on the storage disk that holds your company's persistent data. These database files can contain a customer's personally identifiable information such as credit card and banking details and medical history. And if you are a merchant whose database system is needed to keep sales coming in, you likely have copies of the database in the form of backups ready to be restored quickly to avoid revenue loss. And in healthcare, restoring a database from a backup is critical to preventing a single point of failure that could result in delayed life-saving medical care.

Database backups are critical to avoiding revenue loss and, sometimes, vital in providing timely medical care. Those database backups contain the same information as your primary database serving customers. And in many cases, those database backups can be soft targets without all of the protections and tripwires put in place on the primary database system. The number of people who have access to your database files has expanded with teams responsible for the administration of backups, for example: storage, system, and backup administrators. Since these files are stored on different systems, hackers now have new entry points for breach attempts.


# Transparent Data Encryption for PostgreSQL

Over the last 5 years, PostgreSQL has become a critical component in the most sensitive environments like payment processors, banking, and health care. To ensure their customers’ information doesn't end up for sale on internet forums, Transparent Data Encryption (TDE) is essential for enterprises using Postgres. Using TDE, organizations can enable AES encryption for their Postgres database system. AES encryption is the de facto encryption standard for sensitive applications. TDE means all user data is automatically encrypted when written to disk. And since the database files are encrypted, so are the database backups. The data on disk is unintelligible whether it's on the live database system or in an organization’s backup storage.

Most people comfortably send sensitive data across the internet, trusting internet communication standards used by the largest online retailers, banks, and search engines. TDE helps ensure confidentiality by encrypting data stored in databases, protecting sensitive data even if the database is accessed by unauthorized individuals. It also helps organizations comply with privacy and security regulations, such as GDPR, PCI DSS, and HIPAA, which require the protection of sensitive data. Additionally, investing in transparent data encryption can improve an organization’s reputation by demonstrating a commitment to security and privacy, which increases customer trust. 

The attraction of TDE is clear. Developers, users, admins, and applications can proceed without thinking twice about encryption. However, an encryption key is required toaccess the data, so proper management of those keys is essential. 

As data security concerns rise amongst large businesses that have accelerated their cloud journey, especially those in financial services, data encryption needs to be a best practice. Now organizations can trust storing sensitive data in Postgres using de facto encryption standards and a well-established method in TDE to protect database information.
