# Pertemuan 2: Klasifikasi dengan _Google Earth Engine_
## Unsupervised Classification
### 1. Memilih _image_
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

```javascript
// train cluster on image
var clusterer = ee.Clusterer.wekaKMeans(5).train(training);
```

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

## Supervised Classification
### 1. Memilih _image_
Untuk metode klasifikasi ini, dapat digunakan _image_ dari metode sebelumnya sehingga tidak perlu mengulang _bagian 1_
### 2. Mengumpulkan _training data_
Pada klasifikasi kali ini kita akan menggunakan 7 kelas seperti pada gambar ini:

![class](https://github.com/lindypriyanka/EBA2020/blob/master/6.png)

Seperti metode sebelumnya, kita perlu melakukan _training_ untuk classifier yang akan kita gunakan. Namun dalam metode ini,
kita memberikan sendiri data sampel training yang akan digunakan. Cara membuat data _training sample_ adalah dengan menambahkan ***geometry*** 

![geometry](https://github.com/lindypriyanka/EBA2020/blob/master/7.png)

Kita perlu membuat satu layer geometry untuk setiap kelas dan memberi nama geometry sesuai nama kelas. Kita juga perlu mengganti 
jenis geometry menjadi `featureCollection` pada settings dan menambahkan properties. Properties ini menjadi penanda setiap kelompok kelas ketika nanti disatukan. Silahkan beri nama dan beri nomor dimulai dari 0 untuk setiap kelas.

![feature](https://github.com/lindypriyanka/EBA2020/blob/master/8.png)

Training sample setiap kelas dibuat dengan memberikan _point_ pada daerah yang merepresentasikan kelas tersebut. Semakin banyak point
yang dibuat akan semakin baik hasil yang didapatkan. Point yang diberikan sebisa mungkin harus beragam dan representatif terhadap kelas
tersebut. Selain itu, sebaiknya jumlah point antar kelas sama.

![trainingsample](https://github.com/lindypriyanka/EBA2020/blob/master/9.png)

Setelah training sample seluruh kelas terbuat, kita harus menyatukan data sehingga menjadi satu variable, data disatukan dengan
script berikut:
```javascript
//Merge training sample
var sample = forest.merge(shrubs).merge(agri).merge(building).
              merge(tea).merge(crater);
```

Setelah data di merge, kita dapat melihat hasilnya dengan script
```javascript
print(sample)
```
![merge](https://github.com/lindypriyanka/EBA2020/blob/master/10.png)

Dari hasil tersebut dapat dilihat bahwa terdapat 300 elements dan setiap elemen memiliki properties sesuai dengan yang telah kita
tentukan

### 3. Membuat _training data_
Dari hasil sebelumnya, terlihat bahwa pada kolom properties hanya ada satu nilai. Akan tetapi, dalam klasifikasi, classifier 
memerlukan data dari berbagai jenis band yang dimiliki. Untuk memunculkan data ini, kita harus membuat training data. Cara membuat
training data ini hampir sama dengan metode sebelumnya
```javascript
//Create training from point to image
var bands = ['B2', 'B3', 'B4', 'B5', 'B6', 'B7'];
var training = roicomposite.select(bands).sampleRegions({
    collection: sample, 
    properties: ['LC'], 
    scale: 30
});
```
jika kita visualisasi data dengan command `print` maka akan didapatkan hasil seperti berikut ini

![trainingdata](https://github.com/lindypriyanka/EBA2020/blob/master/11.png)

Berdasarkan data tersebut, dapat terlihat bahwa kolom properties telah menunjukkan nilai dari berbagai band yang kita pilih. Pada kesempatan kali ini kita memilih band 2,3,4,5,6, dan 7.

### 4. _Training classifier_
Langkah selanjutnya adalah untuk men-_train_ classifier kita dengan script berikut. Pastikan bahwa `classProperty` sesuai dengan nama 
properties tiap kelas yang kita buat di awal. `features` diisi dengan training data yang telah kira miliki, dan `inputProperties` 
diisi oleh band-band yang sudah kita pilih diawal
```javascript
//Train the classifier
var classifier02 = ee.Classifier.randomForest().train({
  features: training, 
  classProperty: 'LC', 
  inputProperties: bands
});
```

### 5. Melakukan klasifikasi
Langkah terakhir adalah melakukan klasifikasi kepada gambar yang telah kita pilih diawal. Hasil kemudian dapat divisualisasi dengan
command `Map.addLayer` kita juga dapat memilih palette dari setiap kelas dengan menggunakan kode HTML.

```javascript
//Run the classification
var classified02 = roicomposite.select(bands).classify(classifier02).clip(roi);

Map.addLayer(classified02, {min: 1, max: 5, 
            palette: ['249429', 'FBFE22', 'FA077C','E71313', 
                    '00D1FF', '899187']}, 'classification RF');
Map.setCenter(107.725, -7.3105, 11);
```

Hasil kemudian dapat diexport dengan format GeoTiff seperti metode sebelumnya.
