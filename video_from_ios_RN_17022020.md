# Video is uploaded as an image(thumbnail) from IOS device. (._.)

## Research

I don't know but seem that when get Video URI from IOS, it return a thumbnail not a video. It taken me a day to find out what happen, finally I can not send a absolute path to laravel to get video because I can not receive anything from absolute path.

## My solution

After log hours I decided that I will push video to S3 from frontend (React Native) with aws-sdk. I got a pre-signed URL from backend then use that url to put video to S3 -> call api that upload success to notify backend save data into the database.

### Note:

- I get URI info `FileSystem` package to get thumbnail(IOS seem generate tn for us :~P )
- URI which I put to S3 I use absolute path get from `MediaLibrary.getAssetInfoAsync(id)` library of expo.
- `id` get from URI info by substring from 36 -> &.
- Use `XMLHttpRequest` to run put file because when you use formdata, it will wrap data with content-type `multipart/form-data`. This one just use for IOS, Android I did not test yet.
