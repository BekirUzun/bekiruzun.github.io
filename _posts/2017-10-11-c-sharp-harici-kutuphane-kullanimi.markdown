---
layout: post
title:  "C# Harici Kütüphane Kullanımı: PInvoke"
date:   2017-10-11 22:00:00 -0600
image: /images/post/harici-kutuphane.jpg
tags: c-sharp
excerpt_separator: <!--more-->
---

Geliştirdiğim bir Windows Form uygulamasında bir sorun ile karşılaştım. Windows'un bir ayarını ("Bilgisayar uyandığında şifre sor" ayarı: `Ayarlar > Hesaplar > Oturum açma seçenekleri` ) okumak istiyordum fakat .net framework'ünü kullanarak bir çözüm<!--more--> yolu bulamadım. Biraz araştırmam sonucu Windows'un bir kütüphanesi olan powrprof.dll ile bunun mümkün olduğunu gördüm. Gel gelelim bu C++ ile yazılmış Windows'un içinde bulununan bir kütüphane. Peki bunu C# WinForm uygulamamda nasıl kullanacağım? Cevap: Platform Invoke.

## Platform Invoke Nedir?
Platform Invoke (PInvoke) kısaca; harici dll dosyalarındaki fonksiyonları kullanmamızı sağlayan bir yöntemdir. Bu harici dll dosyalarına **Unmanaged API** da (kodun kontrolü bizim elimizde olmayan) denir. Win32'nin bize sağladığı tonla fonksiyon vardır. Bu fonksiyonlara çeşitli durumlarda ihtiyacımız olabilir.

## Nasıl Yapılır?
Nasıl yapılır kısmını kendi karşılaştığım problem üzerinden anlatacağım. Belki biraz karmaşık olabilir. Daha basit bir örnek için [şurayı](https://docs.microsoft.com/en-us/dotnet/framework/interop/platform-invoke-examples) ziyaret edebilirsiniz.

Öncelikle `Dllimport()` fonksiyonunu kullanabilmek için aşağıdaki namespace'i projemize ekleyelim:

```csharp
using System.Runtime.InteropServices;
```

Daha sonra `Dllimport()` fonksiyonunu kullanalarak istediğimiz fonksiyonu projemize dahil edeceğiz. Fakat öncelikle kullanmak istediğimiz fonksiyonun C# daki karşılığını bulmamız gerekiyor. Bu işleme **Marshaling** deniyor. Fakat bu işlemi manuel yapmak biraz zor. Neyseki abilerimiz bizim için bir çok fonksiyonun C# syntaxını oluşturup, [pinvoke.net](http://pinvoke.net/) sitesinde arşivlemişler. Ben farklı kaynaklardan bulduğum [PowerReadACValueIndex](https://msdn.microsoft.com/en-us/library/windows/desktop/aa372735(v=vs.85).aspx) fonksiyonunu kullandım. Onunda C# sytaxı şu şekilde:

```csharp
[DllImport("powrprof.dll", CharSet = CharSet.Unicode)]
public static extern UInt32 PowerReadACValueIndex(
    IntPtr RootPowerKey,
    ref Guid SchemeGuid,
    ref Guid SubGroupOfPowerSettingsGuid,
    ref Guid PowerSettingGuid,
    ref UInt32 AcValueIndex
);
```
Bu fonksiyondaki IntPtr parametresi pointer, **ref** ile başlayan parametreler ise referans olarak gönderdiğimiz parametrelerdir. Bazı fonksiyonlarda erişmek istediğiniz değerler bu ref ile gönderdiğiniz parametrelere aktarılır. Guid tipindeki parametrelerin değerleri sabit olup, fonksiyonun döndürdüğü UInt32 değeri ise fonksiyonun doğru çalışıp çalışmadığını gösterir. Bu fonksiyonda istediğimiz değer AcValueIndex parametresine atanır. Bu fonksiyonu istediğimiz değeri döndürecek bir fonksiyonun içinde şöyle kullanabiliriz: 

```csharp
public UInt32 GetACValue(Guid scheme, Guid subgroup, Guid setting) {
    UInt32 value = 0;
    PowerReadACValueIndex(IntPtr.Zero, ref scheme, ref subgroup, ref setting, ref value);
    return value;
}
```
Ayrıca bazı fonksiyonlar .Net'in içermediği veri tipte parametreler alabilir. Bu tarz durumlarda yine internetten yararlanıp ilgili veri tipinin **struct** olarak tanımlandığı kod parçalarını bulmamız gerek. Bu iş için de [pinvoke.net](http://pinvoke.net/)'i kullanabiliriz.

Verdiğim örneğinin tam çalışan programı ise bir kaç ek fonksiyon ve geçici sınıf ile şu şekildedir:

```csharp
using System;
using System.Runtime.InteropServices;

namespace PowrprofTest {
    class Program {

        static readonly Guid SUB_NONE = new Guid("fea3413e-7e05-4911-9a71-700331f1c294"); //subgroup
        static readonly Guid CONSOLELOCK = new Guid("0e796bdb-100d-47d6-a2d5-f7d2daa51f51"); //setting
        
        /// <summary>
        /// Aktif güç planının Guid'ini bulmak için kullanılan geçici sınıf
        /// </summary>
        [StructLayout(LayoutKind.Sequential)]
        public class GuidClass {
            public Guid Value;
        }

        /// <summary>
        /// Aktif güç planını bulmak için kullanılan harici Win32 fonksiyonu
        /// </summary>
        [DllImport("powrprof.dll")]
        public static extern UInt32 PowerGetActiveScheme(
            IntPtr UserRootPowerKey,
            ref IntPtr ActivePolicyGuid
        );

        /// <summary>
        /// Aktif güç planındaki bir ayarı okumak için kullanılan harici Win32 fonksiyonu
        /// </summary>
        [DllImport("powrprof.dll", CharSet = CharSet.Unicode)]
        public static extern UInt32 PowerReadACValueIndex(
            IntPtr RootPowerKey,
            ref Guid SchemeGuid,
            ref Guid SubGroupOfPowerSettingsGuid,
            ref Guid PowerSettingGuid,
            ref UInt32 AcValueIndex
        );

        static void Main(string[] args) {
            Guid scheme = GetActiveSchemeGuid(); //aktif güç planının Guid'ini bulalım
            uint value = GetACValue(scheme, SUB_NONE, CONSOLELOCK); //değerimizi okuyalım
            string sleepLock;
            if (value == 1)
                sleepLock = "açık.";
            else
                sleepLock = "kapalı.";
            
            Console.WriteLine("Uyandiktan sonra sifre sorma " + sleepLock);
            Console.ReadLine();
        }

        /// <summary>
        /// Aktif güç planının GUID'ini döndürür
        /// </summary>
        static Guid GetActiveSchemeGuid() {
            IntPtr activeSchemePtr = IntPtr.Zero;
            uint res = PowerGetActiveScheme(IntPtr.Zero, ref activeSchemePtr);
            GuidClass temp = new GuidClass();
            Marshal.PtrToStructure(activeSchemePtr, temp);
            Guid scheme = temp.Value;
            return scheme;
        }      

        /// <summary>
        /// İstenen güç planı ayarının değerini döndürür
        /// </summary>
        static UInt32 GetACValue(Guid scheme, Guid subgroup, Guid setting) {
            UInt32 value = 0;
            PowerReadACValueIndex(IntPtr.Zero, ref scheme, ref subgroup, ref setting, ref value);
            return value;
        }
    }
}

```