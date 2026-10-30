# Modul Praktikum 3: Layout Bersarang (Nested Layout) dan Implementasi Logout

## 1. Tujuan Pembelajaran

Setelah menyelesaikan modul ini, mahasiswa mampu:

* Memahami konsep *Routing* dan *Nested Layouts* pada Next.js App Router.
* Membuat komponen UI persisten (seperti Sidebar atau Navbar) yang tidak di-*render* ulang saat berpindah halaman.
* Mengombinasikan *Server Component* (untuk Layout) dan *Client Component* (untuk interaktivitas tombol).
* Melakukan proses revokasi token (Logout) di API Laravel dan membersihkan *cookies* di sisi klien.

## 2. Prasyarat Praktikum

* Telah menyelesaikan Modul 5 (halaman `/dashboard` sudah terproteksi middleware dan menampilkan data user).
* API Laravel memiliki endpoint `POST /api/logout` yang menggunakan *middleware* `auth:api` untuk menghapus token yang aktif di *database*.

---

## Bagian 1: Membuat Layout Dashboard (Sidebar & Navbar)

Pada arsitektur App Router Next.js, file `layout.tsx` digunakan untuk membungkus halaman (`page.tsx`) di dalam direktori yang sama beserta seluruh sub-direktorinya.

**Langkah 1.1: Membuat File Layout**
Buat file baru bernama `layout.tsx` di dalam folder `app/dashboard/` (sejajar dengan `page.tsx`).

Tambahkan kode berikut:

```tsx
import Link from 'next/link';
import LogoutButton from './components/LogoutButton'; // Kita akan buat komponen ini setelah ini

export default function DashboardLayout({
  children, // Ini adalah representasi dari page.tsx yang sedang aktif
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex min-h-screen bg-gray-100">
      
      {/* Sidebar (Sisi Kiri) */}
      <aside className="w-64 bg-slate-900 text-white flex flex-col">
        <div className="p-6">
          <h2 className="text-2xl font-bold text-blue-400">Sistem Kampus</h2>
        </div>
        
        <nav className="flex-1 px-4 space-y-2 mt-4">
          <Link 
            href="/dashboard" 
            className="block px-4 py-2 rounded transition hover:bg-slate-800"
          >
            Beranda
          </Link>
          <Link 
            href="/dashboard/profil" 
            className="block px-4 py-2 rounded transition hover:bg-slate-800"
          >
            Profil Pengguna
          </Link>
          {/* Tambahkan menu lain di sini */}
        </nav>
      </aside>

      {/* Konten Utama (Sisi Kanan) */}
      <div className="flex-1 flex flex-col">
        
        {/* Navbar Atas */}
        <header className="bg-white shadow-sm border-b h-16 flex items-center justify-between px-8">
          <div className="text-gray-500 font-medium">
            Panel Administrasi
          </div>
          
          {/* Komponen Tombol Logout (Client Component) */}
          <LogoutButton />
        </header>

        {/* Area Konten Dinamis (page.tsx akan masuk ke sini) */}
        <main className="p-8 flex-1 overflow-y-auto">
          {children}
        </main>
        
      </div>
    </div>
  );
}

```

**Penjelasan:** Karena file ini berada di `app/dashboard/layout.tsx`, semua halaman di dalam `/dashboard/*` secara otomatis akan memiliki Sidebar dan Navbar ini. Komponen ini adalah *Server Component* secara default.

**Langkah 1.2: Menyesuaikan Konten Halaman Dashboard**
Karena *styling background* (abu-abu) sekarang diurus oleh `layout.tsx`, mari kita bersihkan sedikit file `app/dashboard/page.tsx` dari modul sebelumnya:

Buka `app/dashboard/page.tsx` dan ubah tag `<main>` menjadi `<div>` biasa:

```tsx
// ... (kode fetch data getUserProfile tetap sama seperti modul sebelumnya)

export default async function DashboardPage() {
  const user = await getUserProfile();
  if (!user) redirect('/login');

  return (
    <div className="bg-white rounded-xl shadow-sm border border-gray-200 p-6">
      <h1 className="text-2xl font-bold text-gray-800 mb-2">
        Selamat Datang, {user.name}!
      </h1>
      <p className="text-gray-600 mb-6">Ini adalah halaman beranda dashboard Anda.</p>
      
      {/* ... (Tabel data user diletakkan di sini) */}
    </div>
  );
}

```

---

## Bagian 2: Implementasi Logout (Client Component)

Karena tombol Logout membutuhkan *event handler* (`onClick`) dan modifikasi *cookies* (menghapus cookie) di browser, ia **wajib** dijadikan *Client Component* (`"use client"`). Itulah mengapa pada *Langkah 1.1* kita memisahkan `<LogoutButton/>` dari `layout.tsx`.

**Langkah 2.1: Membuat Komponen LogoutButton**
Buat folder `components` di dalam `app/dashboard/`. Kemudian buat file `LogoutButton.tsx` (`app/dashboard/components/LogoutButton.tsx`).

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import Cookies from "js-cookie";

export default function LogoutButton() {
  const router = useRouter();
  const [isLoggingOut, setIsLoggingOut] = useState(false);

  const handleLogout = async () => {
    // Hindari double-click
    if (isLoggingOut) return;
    setIsLoggingOut(true);

    try {
      // 1. Ambil token dari cookies browser
      const token = Cookies.get("token");

      if (token) {
        // 2. Beritahu Laravel untuk mencabut (revoke) token ini di database
        await fetch(`${process.env.NEXT_PUBLIC_API_URL}/logout`, {
          method: "POST",
          headers: {
            "Authorization": `Bearer ${token}`,
            "Accept": "application/json",
          },
        });
      }
    } catch (error) {
      console.error("Gagal terhubung ke server saat logout", error);
    } finally {
      // 3. Apapun hasil dari server (berhasil atau gagal), 
      // hapus token dari browser secara paksa agar sesi klien berakhir.
      Cookies.remove("token");
      
      // 4. Redirect ke halaman login dan bersihkan cache router Next.js
      router.push("/login");
      router.refresh(); 
    }
  };

  return (
    <button
      onClick={handleLogout}
      disabled={isLoggingOut}
      className="px-4 py-2 text-sm font-medium text-red-600 bg-red-50 rounded-md hover:bg-red-100 transition disabled:opacity-50"
    >
      {isLoggingOut ? "Keluar..." : "Logout"}
    </button>
  );
}

```

**Langkah 2.2: Uji Coba**

1. Pastikan API Laravel berjalan dan `js-cookie` telah terinstal.
2. Akses `http://localhost:3000/dashboard`.
3. Anda akan melihat Sidebar di sebelah kiri dan Navbar di atas.
4. Klik tombol "Logout" di Navbar.
5. Aplikasi akan mengirim *request* ke Laravel, menghapus *cookies*, dan melempar Anda kembali ke `/login`. Jika Anda mencoba memanggil `/dashboard` lagi di URL browser, *Middleware* dari Modul 2 akan mencegahnya.

---

## 3. Tugas Evaluasi / Praktikum Lanjutan

1. **Navigasi Aktif (Active Link):** Modifikasi file `layout.tsx` (yang akan mengharuskan Anda memisahkan Sidebar menjadi *Client Component* terpisah). Gunakan *hook* `usePathname()` dari `next/navigation` untuk mendeteksi rute mana yang sedang aktif. Jika *user* berada di `/dashboard`, warnai menu "Beranda" menjadi biru gelap.
2. **Sub-halaman Dashboard:** Buat folder `app/dashboard/profil/page.tsx` dan isi dengan komponen sederhana. Navigasikan dari Sidebar ke halaman tersebut dan amati bahwa konten utama berubah, namun Sidebar tidak mengalami *reload*.
