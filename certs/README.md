# 憑證暫存區

從 GoDaddy 下載的 SSL 憑證放這裡,讓 Ansible 部署到 EC2。

⚠️ **這個資料夾的內容(除了 README)都被 .gitignore 排除,不會進 git。**
不要手動把 `.key` 或 `.crt` 檔 commit 上去!

## 目錄結構慣例

每個 domain 一個子資料夾:

```
certs/
├── README.md                  # 這份檔案
├── example.com/
│   ├── certificate.crt        # 主憑證(GoDaddy 給你的 .crt)
│   ├── ca-bundle.crt          # 中繼憑證(GoDaddy 給你的 bundle)
│   └── private.key            # 私鑰(你產 CSR 時自己留的那把)
└── another-domain.com/
    ├── certificate.crt
    ├── ca-bundle.crt
    └── private.key
```

## GoDaddy 下載流程速記

1. GoDaddy → My Products → SSL Certificates → 點選憑證 → Manage
2. **Download** → 選 server type:**Other**(會給你 .crt + ca-bundle)
3. 解壓後會看到:
   - `<random>.crt` ← 主憑證 → 改名成 `certificate.crt`
   - `<random>.pem` 或 `gd_bundle-g2-g1.crt` ← 中繼憑證 → 改名成 `ca-bundle.crt`
4. 私鑰 (`private.key`) 是你**當初產 CSR 時自己保留的那一把**,GoDaddy 不會給你
   - 如果弄丟了,要在 GoDaddy 後台「Re-key」重新產一份

## 憑證有效性快速檢查

部署前可以先在本機驗證:

```bash
cd certs/example.com

# 看憑證資訊(domain、有效期、發行者)
openssl x509 -in certificate.crt -noout -subject -issuer -dates

# 確認憑證和私鑰是同一對(兩個 hash 必須一樣)
openssl x509 -in certificate.crt -noout -modulus | openssl md5
openssl rsa  -in private.key     -noout -modulus | openssl md5
```
