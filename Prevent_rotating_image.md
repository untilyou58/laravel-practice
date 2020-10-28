# Resize the image with Laravel & prevent it from rotating without permission (EXIF is also deleted)

## Introduction

It's nice to have an image posting function in Laravel,

```
"The posted image is large or small .."
"The image rotates without permission after uploading .."
"I heard that the hidden information embedded in the image is dangerous .."
```

We will solve such problems.

The problem of image size can be reduced by specifying it in CSS, but in
CSS you can just change the display and the saved image will remain large.
This puts a load on the database.

In addition, there is `EXIF information` something embedded in the image, such as the information of the smartphone that took the picture, the orientation of the image, and the place where the picture was taken.

We will share how to easily optimize these at the same time.

## Things to do

Follow the procedure below.

① Look at the current EXIF information
② Install Intervention Image (using Composer)
③ Enable Intervention Image (Set Config)
④ Actual image processing with Controller

## Actually try

## 1. First, lets look at the EXIF information of the current image.

Execute the dd() method in the Controller that is currently performing image processing. (By the way, dd() is an abbreviation for 'dump and die')

```php Controller.php
dd($image->exif()); // $image is image data
```

You can now view EXIF information in your browser.
You can see that is displayed in an array and various information such as the model that was shot is stored.

## 2. Install Intervention

It is possible to easily delete this extra EXIF, resize it, and optimize the rotation (image orientation) at the same time `InterventionImage`.

It is a package that supports GD and Imagick, which are PHP image processing libraries, and it is positioned as an option for image processing with Laravel.

You can easily handle GD and Imagick.

Here is the link: [Invertvention](http://image.intervention.io/getting_started/installation#laravel)

Execute the following command to install

```cmd
php composer.phar require intervention/image
```

## 3. Make Intervention Image available

Add the following descrition to Config/app.php

It is to add providers and aliases is two places of.

```php Config/app.php
'providers' => [
    Intervention\Image\ImageServiceProvider::class,
],

~ommision~

'aliases' => [
    'Image' => Intervention\Image\Facades\Image::class,
],
```

In addition, add driver settings. (Execute the following command)

```cmd
php artisan vendor:publish --provider="Intervention\Image\ImageServiceProviderLaravelRecent"
```

Finally, I messed with the settings in Config, so clear the cache. (Execute the following command)

```cmd
php artisan config:clear
```

## 4.Actually process with Controller

### Declare to use the façade

```php Controller
use Intervention\Image\Facades\Image; // Use image facade
use Illuminate\Support\Facades\Storage; // Use storage facade
```

### Perform image processing

InterventionImage can do a lot of things, but
here's
an example of resizing while maintaining the aspect ratio that you'll often use and converting files to jpg format

```php
$resized_image = Image::make($posted_image)->fit(640, 360)->encode('jpg');
```

`$posted_image` is the unprocessed image, and `make()` loads the unprocessed image into InterventionImage.
fit () is a method that specifies the width and height and trims if the aspect ratio does not match.

For simple resizing, resize() is fine, but I think this fit() is convenient because the image will be distorted.

* For comprehensive image processing with Intervention Image, refer to this site.

[https://blog.capilano-fw.com/?p=1574#_resize](https://blog.capilano-fw.com/?p=1574#_resize)


## Image rotation and processing of EXIF ​​information

Finally, there is the problem that the image of the main subject (?) Rotates freely and the processing of EXIF ​​information.

This `orientate()` use.

```php
$resized_image->orientate()->save();
```

This will automatically rotate and delete the EXIF information.

Note that the process will not be reflected without `->save()`.

## Save the file

Since the `store` method cannot be used for saving a file (an error that GD does not support it)
Use `Storage::put` of Laravel function (facade)

```php
Storage::put('public/image/' . $image_name, $resized_image); 
```

You should now have a nice size, the image is in the correct orientation, and you should be able to post an image with unnecessary embed information removed!

[Source](https://qiita.com/paleo_engineer/items/8d487c6a5683ca1be3da)
