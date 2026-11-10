# Maintainer: phonecam contributors
# Builds the git version. The release PKGBUILD uses a tarball instead of
# "git+..." in source=().
pkgname=phonecam-git
pkgver=1.0.0
pkgrel=1
pkgdesc="Phone camera and Lovense Cam overlay for OBS"
arch=('any')
url="https://github.com/OWNER/phonecam"
license=('MIT')
depends=('python' 'python-websockets' 'python-aiohttp' 'python-cryptography'
         'qrencode' 'p7zip')
optdepends=('obs-studio: capture and scene management'
            'google-chrome: Lovense Cam extension'
            'chromium: Lovense Cam extension')
provides=('phonecam')
conflicts=('phonecam')
options=('!strip')
source=("git+${url}.git")
sha256sums=('SKIP')

package() {
  cd "$srcdir/phonecam"

  install -Dm755 phonecam "$pkgdir/usr/bin/phonecam"
  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
  install -Dm644 NOTICE "$pkgdir/usr/share/licenses/$pkgname/NOTICE"

  install -d "$pkgdir/usr/share/phonecam"
  cp -a widget "$pkgdir/usr/share/phonecam/widget"
}
