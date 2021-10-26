# ビデオカメラで撮影した動画がOneDriveで見れない件
## きっかけ
Googleフォト無制限利用が終了してしまったので新たに動画をアップロードする先を考えていた。
office 365 soloを契約していたので1TB分のOneDriveを使おうと思いビデオカメラで撮影したファイルをアップロード。
しかし、再生しようとすると「ストリームの再生に問題が発生しました」と出てしまう。
iPhoneで撮影した動画であれば正常に再生できるため、再生できる動画形式を探して旅に出ることとなった。

### OneDriveでの再生
OneDriveにアップロードされた動画をダウンロードせずに再生する場合、ストリーミング配信になる。
つまり再生時に(もしかしたらアップロード時かもしれないが)ストリーミング再生可能なフォーマット(dash or hls)に変換している。

## 拡張子
ビデオカメラで撮影した動画が.MTS、iPhoneで撮影した動画は.MOVということで拡張子に差があることが分かった。
では.MTSから.mp4に変換してしまおうということで、ffmpegで以下のコマンドを実行した。
```
ffmpeg -i video.MTS -vcodec copy -acodec copy video.mp4
```
このコマンドではMTSの動画ファイルからMP4のファイルに変換しているのだが、
`-vcodec copy -acodec copy`で映像・音声のコーデックをそのままに変換している。

これで変換したmp4ファイルをローカルで再生できるか確認する→問題なく再生
OneDriveに移動して再生確認→「ストリームの再生に問題が発生しました」

ということはコーデックに問題があるのだろうか。

## コーデック
拡張子の問題ではないのであればコーデックの問題ということでまずはffprobeで確認する

MTS
```
Input #0, mpegts, from '00001.MTS':
  Duration: 00:00:37.06, start: 1.033367, bitrate: 17190 kb/s
  Program 1
  Stream #0:0[0x1011]: Video: h264 (High) (HDMV / 0x564D4448), yuv420p(top first), 1920x1080 [SAR 1:1 DAR 16:9], 29.97 fps, 59.94 tbr, 90k tbn, 59.94 tbc
  Stream #0:1[0x1100]: Audio: ac3 (AC-3 / 0x332D4341), 48000 Hz, stereo, fltp, 256 kb/s
  Stream #0:2[0x1200]: Subtitle: hdmv_pgs_subtitle ([144][0][0][0] / 0x0090), 1920x1080
```
MOV
```
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '.\IMG_7294.MOV':
  Metadata:
    major_brand     : mp42
    minor_version   : 0
    compatible_brands: isommp42
    creation_time   : 2020-12-11T01:05:46.000000Z
  Duration: 00:00:09.59, start: 0.000000, bitrate: 3866 kb/s
  Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709), 1920x1080 [SAR 1:1 DAR 16:9], 3751 kb/s, 59.94 fps, 59.94 tbr, 60k tbn, 119.88 tbc (default)
    Metadata:
      creation_time   : 1970-01-01T00:00:00.000000Z
      handler_name    : ISO Media file produced by Google Inc.
      vendor_id       : [0][0][0][0]
  Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 128 kb/s (default)
    Metadata:
      creation_time   : 1970-01-01T00:00:00.000000Z
      handler_name    : ISO Media file produced by Google Inc.
      vendor_id       : [0][0][0][0]
```

比較したところ音声コーデックが`AC3`と`AAC`という違いがあった。
dashであれば`AC3`も使える認識だがmp4として載せられなかったのだろう。
ということで音声コーデックだけ書き換える方針に変更
```
ffmpeg -i video.MTS -vcodec copy  video.mp4
```
これで変換したmp4ファイルをローカルで再生できるか確認する→問題なく再生
OneDriveに移動して再生確認→問題なく再生

これで解決！

## 大量にファイルあるんですよ
とはいえ150ファイルあるのでスクリプトを書くことにする
```
$index = 0
$array = Get-ChildItem *.MTS -Name
$count = $array.count
while ($index -le $count) {
    $newfile = -join("..\mp4\",($array[$index] -replace "MTS","mp4"))
    ffmpeg -i $array[$index] -vcodec copy $newfile
    $index += 1
}
```
対象となるMTSファイルの名前を配列に格納し、ループして取り出す奴をやってます
書き方がスマートかどうかは知りません（おしえてください）

こちらで書き出して無事完了。（容量不足で中断したりしましたが）

## まとめ
* OneDriveから動画ファイルを見る時はストリーミング配信になりフォーマット変換される
* AC3だとうまくいかないのでAACに変換してあげる必要あり
* Googleフォトさんは裏でやってくれてた