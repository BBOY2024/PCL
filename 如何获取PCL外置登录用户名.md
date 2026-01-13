# 如何获取 PCL 外置登录的用户名

## 问题
您有一个软件想要直接使用 PCL 外置登录产生的用户名。

## 加密方式

PCL 使用 **DES (Data Encryption Standard)** 加密算法来保护注册表中的敏感数据。

### 加密参数
- **算法**: DES
- **密钥**: 固定的 8 位密钥 `"00000000"`（开源版本中）
- **初始化向量 (IV)**: `"87160295"`
- **编码**: UTF-8
- **输出格式**: Base64

### 代码实现
加密和解密的代码位于 `Plain Craft Launcher 2/Modules/ModSecret.vb`：

```vb
' 加密函数
Friend Function SecretEncrypt(SourceString As String, Optional Key As String = "") As String
    Key = SecretKeyGet(Key)  ' 返回 "00000000"
    Dim btKey As Byte() = Encoding.UTF8.GetBytes(Key)
    Dim btIV As Byte() = Encoding.UTF8.GetBytes("87160295")
    Dim des As New DESCryptoServiceProvider
    Using MS As New MemoryStream
        Dim inData As Byte() = Encoding.UTF8.GetBytes(SourceString)
        Using cs As New CryptoStream(MS, des.CreateEncryptor(btKey, btIV), CryptoStreamMode.Write)
            cs.Write(inData, 0, inData.Length)
            cs.FlushFinalBlock()
            Return Convert.ToBase64String(MS.ToArray())
        End Using
    End Using
End Function

' 解密函数
Friend Function SecretDecrypt(SourceString As String, Optional Key As String = "") As String
    Key = SecretKeyGet(Key)  ' 返回 "00000000"
    Dim btKey As Byte() = Encoding.UTF8.GetBytes(Key)
    Dim btIV As Byte() = Encoding.UTF8.GetBytes("87160295")
    Dim des As New DESCryptoServiceProvider
    Using MS As New MemoryStream
        Dim inData As Byte() = Convert.FromBase64String(SourceString)
        Using cs As New CryptoStream(MS, des.CreateDecryptor(btKey, btIV), CryptoStreamMode.Write)
            cs.Write(inData, 0, inData.Length)
            cs.FlushFinalBlock()
            Return Encoding.UTF8.GetString(MS.ToArray())
        End Using
    End Using
End Function
```

## 用户名存储位置

用户名存储在 Windows 注册表中，路径为：
```
HKEY_CURRENT_USER\Software\PCLDebug\
```

### 可用的用户名键

根据登录类型，用户名存储在不同的注册表键中：

#### 统一通行证 (Nide8)
- **CacheNideName** - 显示名称（游戏内显示的名字）
- **CacheNideUsername** - 用户账号名

#### Authlib-Injector
- **CacheAuthName** - 显示名称（游戏内显示的名字）
- **CacheAuthUsername** - 用户账号名

**建议使用 `CacheNideName` 或 `CacheAuthName`**，因为这通常是游戏内显示的名字。

## 如何在您的软件中获取用户名

### 方法 1: 使用 C# 代码

```csharp
using System;
using System.IO;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Win32;

public class PCLUserNameReader
{
    // DES 解密函数
    private static string DecryptDES(string encryptedString)
    {
        byte[] key = Encoding.UTF8.GetBytes("00000000");
        byte[] iv = Encoding.UTF8.GetBytes("87160295");
        
        using (DESCryptoServiceProvider des = new DESCryptoServiceProvider())
        using (MemoryStream ms = new MemoryStream())
        {
            byte[] inputData = Convert.FromBase64String(encryptedString);
            using (CryptoStream cs = new CryptoStream(ms, des.CreateDecryptor(key, iv), CryptoStreamMode.Write))
            {
                cs.Write(inputData, 0, inputData.Length);
                cs.FlushFinalBlock();
                return Encoding.UTF8.GetString(ms.ToArray());
            }
        }
    }
    
    // 从注册表读取并解密用户名
    public static string GetNideUserName()
    {
        try
        {
            using (RegistryKey key = Registry.CurrentUser.OpenSubKey(@"Software\PCLDebug"))
            {
                if (key != null)
                {
                    string encryptedName = key.GetValue("CacheNideName") as string;
                    if (!string.IsNullOrEmpty(encryptedName))
                    {
                        return DecryptDES(encryptedName);
                    }
                }
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"读取失败: {ex.Message}");
        }
        return null;
    }
    
    // 获取 Authlib-Injector 用户名
    public static string GetAuthlibUserName()
    {
        try
        {
            using (RegistryKey key = Registry.CurrentUser.OpenSubKey(@"Software\PCLDebug"))
            {
                if (key != null)
                {
                    string encryptedName = key.GetValue("CacheAuthName") as string;
                    if (!string.IsNullOrEmpty(encryptedName))
                    {
                        return DecryptDES(encryptedName);
                    }
                }
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"读取失败: {ex.Message}");
        }
        return null;
    }
    
    // 自动检测并返回可用的用户名
    public static string GetUserName()
    {
        string username = GetNideUserName();
        if (!string.IsNullOrEmpty(username))
            return username;
        
        username = GetAuthlibUserName();
        if (!string.IsNullOrEmpty(username))
            return username;
        
        return null;
    }
}

// 使用示例
class Program
{
    static void Main()
    {
        string username = PCLUserNameReader.GetUserName();
        if (username != null)
        {
            Console.WriteLine($"用户名: {username}");
        }
        else
        {
            Console.WriteLine("未找到用户名");
        }
    }
}
```

### 方法 2: 使用 Python 代码

```python
import winreg
import base64
from Crypto.Cipher import DES
from Crypto.Util.Padding import unpad

def decrypt_des(encrypted_string):
    """解密 DES 加密的字符串"""
    key = b'00000000'
    iv = b'87160295'
    
    cipher = DES.new(key, DES.MODE_CBC, iv)
    encrypted_data = base64.b64decode(encrypted_string)
    decrypted_data = cipher.decrypt(encrypted_data)
    
    # 移除 PKCS7 padding
    try:
        decrypted_data = unpad(decrypted_data, DES.block_size)
    except ValueError:
        pass
    
    return decrypted_data.decode('utf-8')

def get_username_from_registry(key_name):
    """从注册表读取并解密用户名"""
    try:
        key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\PCLDebug")
        encrypted_name, _ = winreg.QueryValueEx(key, key_name)
        winreg.CloseKey(key)
        
        if encrypted_name:
            return decrypt_des(encrypted_name)
    except Exception as e:
        print(f"读取失败: {e}")
    return None

def get_pcl_username():
    """获取 PCL 外置登录的用户名"""
    # 先尝试 Nide8
    username = get_username_from_registry("CacheNideName")
    if username:
        return username
    
    # 再尝试 Authlib-Injector
    username = get_username_from_registry("CacheAuthName")
    if username:
        return username
    
    return None

# 使用示例
if __name__ == "__main__":
    username = get_pcl_username()
    if username:
        print(f"用户名: {username}")
    else:
        print("未找到用户名")
```

**注意**: Python 需要安装 `pycryptodome` 库：
```bash
pip install pycryptodome
```

### 方法 3: 使用 PowerShell 脚本

```powershell
# DES 解密函数
function Decrypt-DES {
    param(
        [string]$EncryptedString
    )
    
    $key = [System.Text.Encoding]::UTF8.GetBytes("00000000")
    $iv = [System.Text.Encoding]::UTF8.GetBytes("87160295")
    
    $des = New-Object System.Security.Cryptography.DESCryptoServiceProvider
    $des.Key = $key
    $des.IV = $iv
    $des.Mode = [System.Security.Cryptography.CipherMode]::CBC
    
    $encryptedBytes = [System.Convert]::FromBase64String($EncryptedString)
    $decryptor = $des.CreateDecryptor()
    
    $ms = New-Object System.IO.MemoryStream
    $cs = New-Object System.Security.Cryptography.CryptoStream($ms, $decryptor, [System.Security.Cryptography.CryptoStreamMode]::Write)
    
    $cs.Write($encryptedBytes, 0, $encryptedBytes.Length)
    $cs.FlushFinalBlock()
    
    $decryptedBytes = $ms.ToArray()
    $decryptedString = [System.Text.Encoding]::UTF8.GetString($decryptedBytes)
    
    $cs.Close()
    $ms.Close()
    
    return $decryptedString
}

# 从注册表获取用户名
function Get-PCLUserName {
    $regPath = "HKCU:\Software\PCLDebug"
    
    # 尝试读取 Nide8 用户名
    try {
        $encryptedName = Get-ItemPropertyValue -Path $regPath -Name "CacheNideName" -ErrorAction Stop
        if ($encryptedName) {
            return Decrypt-DES -EncryptedString $encryptedName
        }
    } catch {
        Write-Host "未找到 Nide8 用户名"
    }
    
    # 尝试读取 Authlib 用户名
    try {
        $encryptedName = Get-ItemPropertyValue -Path $regPath -Name "CacheAuthName" -ErrorAction Stop
        if ($encryptedName) {
            return Decrypt-DES -EncryptedString $encryptedName
        }
    } catch {
        Write-Host "未找到 Authlib 用户名"
    }
    
    return $null
}

# 使用示例
$username = Get-PCLUserName
if ($username) {
    Write-Host "用户名: $username"
} else {
    Write-Host "未找到用户名"
}
```

## 注意事项

1. **开源版本限制**: 上述代码基于 PCL 的开源版本。开源版本使用固定密钥 `"00000000"`。正式版本可能使用不同的密钥生成方式。

2. **安全性**: DES 算法已被认为不够安全，但由于这只是本地数据保护，风险相对较低。

3. **权限**: 您的软件需要有读取当前用户注册表的权限。

4. **检查空值**: 如果用户尚未登录过，相应的注册表键可能不存在或为空。

5. **选择正确的键**: 
   - 如果您的用户使用统一通行证，读取 `CacheNideName`
   - 如果您的用户使用 Authlib-Injector，读取 `CacheAuthName`
   - 为了兼容性，可以两个都尝试读取

6. **UUID vs 显示名称**:
   - `CacheNideName` / `CacheAuthName` - 游戏内显示的名字（推荐使用）
   - `CacheNideUuid` / `CacheAuthUuid` - 玩家的 UUID（唯一标识符）
   - `CacheNideUsername` / `CacheAuthUsername` - 登录账号名

## 相关注册表键说明

| 键名 | 说明 | 用途 |
|------|------|------|
| CacheNideName | 统一通行证显示名称 | **游戏内显示的名字（推荐）** |
| CacheNideUsername | 统一通行证账号名 | 登录账号 |
| CacheNideUuid | 统一通行证 UUID | 玩家唯一标识 |
| CacheAuthName | Authlib 显示名称 | **游戏内显示的名字（推荐）** |
| CacheAuthUsername | Authlib 账号名 | 登录账号 |
| CacheAuthUuid | Authlib UUID | 玩家唯一标识 |

## 完整流程

1. 打开注册表路径 `HKEY_CURRENT_USER\Software\PCLDebug\`
2. 读取 `CacheNideName` 或 `CacheAuthName` 的值（Base64 编码的加密字符串）
3. 使用 DES 解密（密钥: `"00000000"`, IV: `"87160295"`）
4. 得到明文用户名

## 示例

假设注册表中 `CacheNideName` 的值为：
```
kRz8YnBmGXk=
```

解密后得到：
```
Steve
```

这就是用户的游戏内显示名称。

## 总结

您只需要：
1. 读取注册表 `HKEY_CURRENT_USER\Software\PCLDebug\CacheNideName` 或 `CacheAuthName`
2. 使用 DES 算法解密（密钥: `"00000000"`, IV: `"87160295"`）
3. 得到用户名

使用上面提供的代码示例可以直接在您的软件中实现这个功能。
