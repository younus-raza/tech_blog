
## Setup Images, Logo and Favicon using imagicmagic
```sh
brew install imagemagick
# Make 16x16 favicon 
cd assets/images
magick YounusRaza_Logo_40x40.png -resize 16x16 -filter Lanczos favicon.ico

```