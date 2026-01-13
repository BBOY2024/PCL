# Third-Party External Login Data Storage Location

## Summary

This document provides information about where Plain Craft Launcher 2 (PCL) stores third-party external login data.

## Storage Location

All encrypted login data is stored in the Windows Registry at:
```
HKEY_CURRENT_USER\Software\PCLDebug\
```

## Supported Third-Party Login Methods

### 1. Nide8 Unified Passport (统一通行证)
Registry keys:
- `LoginNideEmail` - Email address (encrypted)
- `LoginNidePass` - Password (encrypted)
- `CacheNideAccess` - Access token (encrypted)
- `CacheNideClient` - Client token (encrypted)
- `CacheNideUuid` - User UUID (encrypted)
- `CacheNideName` - Display name (encrypted)
- `CacheNideUsername` - Username (encrypted)
- `CacheNideServer` - Server address (encrypted)

### 2. Authlib-Injector
Registry keys:
- `LoginAuthEmail` - Email address (encrypted)
- `LoginAuthPass` - Password (encrypted)
- `CacheAuthAccess` - Access token (encrypted)
- `CacheAuthClient` - Client token (encrypted)
- `CacheAuthUuid` - User UUID (encrypted)
- `CacheAuthName` - Display name (encrypted)
- `CacheAuthUsername` - Username (encrypted)
- `CacheAuthServerServer` - Server address (encrypted)
- `CacheAuthServerName` - Server name (encrypted)
- `CacheAuthServerRegister` - Registration URL (encrypted)

## Implementation Details

### Key Files

1. **ModSetup.vb** (`Plain Craft Launcher 2/Pages/PageSetup/ModSetup.vb`)
   - Lines 59-83: Setup entry definitions
   - All login-related settings are marked with `Source:=SetupSource.Registry` and `Encoded:=True`

2. **ModBase.vb** (`Plain Craft Launcher 2/Modules/Base/ModBase.vb`)
   - Line 590: `ReadReg()` - Read from registry
   - Line 601: `WriteReg()` - Write to registry
   - Line 614: `HasReg()` - Check if key exists
   - Line 620: `DeleteReg()` - Delete registry key

3. **Login Pages**
   - Nide8: `Plain Craft Launcher 2/Pages/PageLaunch/PageLoginNide.xaml.vb`
   - Authlib: `Plain Craft Launcher 2/Pages/PageLaunch/PageLoginAuth.xaml.vb`

### Encryption

- All sensitive data is encrypted using `SecretEncrypt()` function
- Encryption key: `"PCL" + Identify` (where Identify is user-specific)
- Decryption uses `SecretDecrypt()` function

### Data Format

- **Multiple accounts**: Separated by `¨` character
  - Example: `account1¨account2¨account3`
- **Reading**: Split using `.Split("¨")`
- **Selection**: Access by index

## Security Features

1. All sensitive data (emails, passwords, tokens) are encrypted before storage
2. Encryption key is bound to user identifier for additional security
3. Data stored in user-level registry, inaccessible to other users
4. Registry location is under current user hive (HKEY_CURRENT_USER)

## How to Access the Data

### Method 1: Registry Editor
1. Press `Win + R`
2. Type `regedit` and press Enter
3. Navigate to `HKEY_CURRENT_USER\Software\PCLDebug\`
4. View stored keys (data is encrypted)

### Method 2: Through PCL Launcher
- Use the login interface to manage accounts
- Control password saving with "Remember Password" option

### Method 3: Programmatically
```vb
' Read Nide8 email
Dim email = Setup.Get("LoginNideEmail")

' Read Authlib email
Dim authEmail = Setup.Get("LoginAuthEmail")

' Clear cached access token (force re-login)
Setup.Set("CacheNideAccess", "")
Setup.Set("CacheAuthAccess", "")
```

## Important Notes

1. **Data Migration**: When switching computers, you'll need to log in again since encryption keys are environment-specific
2. **Security**: Although data is encrypted, regular password changes are still recommended
3. **Debug Mode**: Registry path contains "PCLDebug" which may be from development naming
4. **Password Management**: Disable "Remember Password" if you don't want credentials saved

## Conclusion

PCL stores third-party external login data in the Windows Registry at `HKEY_CURRENT_USER\Software\PCLDebug\`. All sensitive information is encrypted. Two main third-party login methods are supported:
1. **Nide8 Unified Passport** - Uses `LoginNide*` and `CacheNide*` keys
2. **Authlib-Injector** - Uses `LoginAuth*` and `CacheAuth*` keys

User data security is ensured through encryption and user-level registry access control.
