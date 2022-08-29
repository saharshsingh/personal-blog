# Personal Blog

[http://saharsh.org](http://saharsh.org)  
[http://www.saharsh.org](http://www.saharsh.org)  

## Build Instructions

### Test locally

    hugo server --bind=<local-ip> --baseURL="http://<local-ip>"

### Build for production

    rm -r public; hugo

**Considerations**
1. When building for production, build for `//saharsh.org` AND `//www.saharsh.org` (Change `baseURL` in `config.yaml`)
1. Invalidate CDN cache for both `saharsh.org/` and `www.saharsh.org/` after updating cloud storage buckets
