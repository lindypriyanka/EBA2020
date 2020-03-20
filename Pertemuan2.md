# Pertemuan 2: Klasifikasi dengan _Google Earth Engine_
## Unsupervised Classification
### 1. Memunculkan _image_
Seperti yang telah dilakukan dua minggu lalu, tahap pertama sebelum melakukan analisis adalah memunculkan _image_ yang kita inginkan.
Kali ini, kita juga menggunakan data dari Landsat 8. Maka, data bisa di _import_ ke dalam _script_ dengan mencari "Landsat 8" kemudian
menekan _import_. Silahkan mengganti nama _variable_ untuk koleksi data ini menjadi ***L8***
![importL8](https://github.com/lindypriyanka/EBA2020/blob/master/1.png)

Setelah koleksi data dipilih, kita bisa mulai memilih foto yang ingin digunakaan. Kali ini, kita akan memilih foto berdasarkan region,
tanggal, dan ada atau tidaknya awan. Silahkan pilih region menggunakan pilihan _Draw a rectangle_ dan tentukan daerah yang ingin
kita analisis, dalam contoh ini daerah Gunung Papandayan. Kemudian namakan variable yang muncul ***roi***
![filter ROI](https://github.com/lindypriyanka/EBA2020/blob/master/2.png)

Kemudian kita akan menggunakan _script_ dibawah ini untuk menyaring foto dengan awan yang minimum
```javascript
// Function to mask clouds using the quality band of Landsat 8.
var maskL8 = function(image) {
  var qa = image.select('BQA');
  var mask = qa.bitwiseAnd(1 << 4).eq(0);
  return image.updateMask(mask);
}
```
Terakhir, kita gabungkan seluruh _filter_, yaitu _filter_ region, tanggal, dan awan dengan script ini:
```javascript
// Map the function over one year of Landsat 8 TOA data and take the median.
var composite = L8
    .filterDate('2019-01-01', '2019-12-31')
    .map(maskL8)
    .median();
var roicomposite = composite.clip(roi);
```

Kita bisa melihat hasil peta yang kita dapat setelah _filter_ ini dengan visualisasi menggunakan `Map.addLayer`
```javascript
Map.addLayer(roicomposite, {bands: ['B4', 'B3', 'B2'],
            min:0, max: 0.2}, 'True colour image');
```
![truecolorimage](https://github.com/lindypriyanka/EBA2020/blob/master/3.png)

Tahap terakhir untuk mempersiapkan foto adalah dengan merubah seluruh proses yang telah kita lakukan menjadi sabtu _variable geometry_
```javascript
// use the bounding box of a Landsat-8 image
var region = roicomposite.geometry();
```
### 2. Membuat _training sample_
Setelah foto yang akan dilakukan analisis didapatkan, kita bisa memulai proses klasifikasi dengan metode _unsupervised_. Dalam proses 
klasifikasi diperlukan _training sample_ sebagai bahan ajar untuk _classifier_. Training untuk classifier dilakukan pertama-tama 
dengan mendefinisikan daerah yang akan di training menggunakan script ini
```javascript
// training region in the full scene
var training = roicomposite.sample({
  region: region,
  scale: 30,
  numPixels: 5000
});
```
Pada script ini, `region` menggambarkan daerah yang menjadi sumber _training_, dan `scale` menggambarkan resolusi, yang kali ini digunakan
30/30m sesuai dengan resolusi Landsat 8. Setelah itu, kita bisa menjalankan traning menggunakan script berikut. Pada tahap ini juga kita
bisa memilih berapa jumlah kelas yang kita inginkan. Contoh ini menggunakan 5 kelas, tapi silahkan mencoba untuk berbagai jumlah kelas.

// train cluster on image
var clusterer = ee.Clusterer.wekaKMeans(5).train(training);

### 3. Menjalankan klasifikasi
Setelah classifier berhasil di _training_ kita bisa memjalankan klasifikasi dengan script berikut dan hasilnya dapat kemudian dilihat
dengan visualisasi menggunakan `Map.addLayer`
```javascript
// cluster the complete image
var classresult = roicomposite.cluster(clusterer);

// Display the clusters with random colors.
Map.addLayer(classresult.clip(roi).randomVisualizer(),{} , 'classresult'
);
```

Berikut adalah hasil klasifikasi yang didapat
![resultunsupervised](https://github.com/lindypriyanka/EBA2020/blob/master/4.png)

### 4. Exporting image
Terakhir, kita dapat meng-_export_ hasil klasifikasinya dengan format GeoTiff dengan script berikut
```javascript
//export map
Export.image.toDrive({
  image: classresult.randomVisualizer(),
  fileNamePrefix:"unsupervised",
  region: roi,
  scale: 30,
  crs:'EPSG:32750',
  maxPixels:1e13
})
```
Dalam script ini, kita meng-_export_ image ke _Google Drive_ masing-masing. kolom `image` menggambarkan hasil yang ingin kita _export_
beserta pengaturan visualisasinya, `fileNamePrefix` merupakan nama file yang ingin kita buat, `region` adalah daerah dari hasil klasifikasi
yang ingin kita _export_, dan `scale` adalah resolusi dari foto. 

Setelah script di _run_, hasil akan muncul pada kolom ***task***. Klik tombol ***run*** yang muncul di samping hasil sehingga seharusnya
muncul _window_ ini
![export](https://github.com/lindypriyanka/EBA2020/blob/master/5.png)
kemudian kita bisa merubah kembali nama file, scale, dan lokasi dari file yang akan di export. Jika seluruhnya sudah sesuai, klik _run_ dan hasil akan muncul di folder GoogleDrive yang telah dipilih
