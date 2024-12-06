# XXE

**Tags**: [[Tech]] [[Cybersecurity]]

[[XML]] External Entity (XXE) is a vulnerability that takes advantage of how [[XML]] parsers handle external entities when loaded.

For example this [[XML]] code when parsed, inserts the file into the `data` field. This is intended behavior but it can be used to read sensitive files if there is no proper sanitation. 

```xml
<?xml version="1.0"?> 
<!DOCTYPE data [ <!ENTITY xxe SYSTEM "file:///path/to/some/file"> ]> 
<data>&xxe;</data>
```

# References:

- [XXE Portswigger](https://portswigger.net/web-security/xxe)
