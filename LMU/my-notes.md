LMU
golder rule :
- hanya bisa dibatik
- hanya bisa ekonomi ke bisnis
- depart date (2-1 jam sblm flight)
- coupon status : (cuman bisa OK dan checkin)
- avalabiilty (ada bisnis class dan seat nya gak penuh)
- flight status gak boleh : held, close, PDC

- check status user : a. kalo blm checkin (upgrade class -> bisnis), dan issued EMD by sabre
b. kalo udah checkin : i. check offload (dari checkin menjadi gak ckin)
ii. check seat lama (apakah masih hold atau udah di reserve)
iii. proses lanjut ke point a
notes, kalo point a1

- LMU bisa gratis dan bisa bayar
- gratis ada quota nya: ada di LMU quota
- max quota special : ada relasi ke period type = misal dalam satu bulan (enum 1 misal) hanya ada 100 quota yg di update
- pattern nya yg as is : max quota 2x : max quota special

terkait project baru :
- LMU gratis : selain quota hanya bisa untuk membership BC
orang A -> BC (bisa free upgrade)
orang B -> yg ngikut A (bisa free upgrade juga)
kalo hanya B yg upgrade, bakal ada split PNR (tapi dalam masih dlm satu order)

- LMU full bayar (as is)
- LMU bayar dgn discount : tanya apakah ada quota nya


flow gede di LMU
Details
Prepare
Payment


dict
- VCR (virtual coupon record) : 1 : 1 ticket number (penumpang nya)
- VCR : segment = 1:m
- PNR : VCR = 1 : m
- VCR punya banyak coupon (segment) | segment per flight (stop)
- round flight -> beda VCR
- flight status
- handle harga : sudah ada di db kita
- trx sudah di record di db sendiri : table LMU order
- ticker number (buat flight aja)
- EMD number (kepake waktu pake addons, anciliary, dll)
- BC dan non BC : waktu order nya di mana
- uang dari hasil upgrade LMU gak bisa di refund

- PAX : ADT (adult), INF (infant), CHD (child)

- kalo LMU free, waktu di prepare udah nulis order success