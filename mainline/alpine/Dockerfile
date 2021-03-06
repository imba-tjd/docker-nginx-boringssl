FROM alpine:3.12

ENV NGINX_VERSION=1.19.3

ARG BUILD_GEOIP2=false
ARG BUILD_DEBUG=false

RUN GPG_KEYS=B0F4253373F8F6F510D42178520A9993A1C052F8 \
	&& CONFIG="\
		--prefix=/etc/nginx \
		--sbin-path=/usr/sbin/nginx \
		--modules-path=/usr/lib/nginx/modules \
		--conf-path=/etc/nginx/nginx.conf \
		--error-log-path=/var/log/nginx/error.log \
		--http-log-path=/var/log/nginx/access.log \
		--pid-path=/var/run/nginx.pid \
		--lock-path=/var/run/nginx.lock \
		--http-client-body-temp-path=/var/cache/nginx/client_temp \
		--http-proxy-temp-path=/var/cache/nginx/proxy_temp \
		--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
		--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
		--http-scgi-temp-path=/var/cache/nginx/scgi_temp \
		--user=nginx \
		--group=nginx \
		--with-pcre-jit \
		--with-http_ssl_module \
		--with-http_realip_module \
		--with-http_addition_module \
		--with-http_sub_module \
		--with-http_dav_module \
		--with-http_flv_module \
		--with-http_mp4_module \
		--with-http_gunzip_module \
		--with-http_gzip_static_module \
		--with-http_random_index_module \
		--with-http_secure_link_module \
		--with-http_stub_status_module \
		--with-http_auth_request_module \
		--with-http_xslt_module=dynamic \
		--with-http_image_filter_module=dynamic \
		--with-http_perl_module=dynamic \
		--with-threads \
		--with-stream=dynamic \
		--with-stream_ssl_module \
		--with-stream_ssl_preread_module \
		--with-stream_realip_module \
		--with-http_slice_module \
		--with-mail \
		--with-mail_ssl_module \
		--with-compat \
		--with-file-aio \
		--with-http_v2_module \
		--add-dynamic-module=/usr/src/ngx_headers_more \
		--add-dynamic-module=/usr/src/ngx_brotli \
	" \
	&& CMPL_FLAGS="-O2 -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2" \
	&& CC_OPTS=--with-cc-opt="$CMPL_FLAGS -I/usr/src/boringssl/.openssl/include" \
	&& LD_OPTS=--with-ld-opt="-Wl,-z,relro -Wl,-z,now -Wl,--as-needed -L/usr/src/boringssl/.openssl/lib" \
	&& addgroup -S nginx \
	&& adduser -D -S -h /var/cache/nginx -s /sbin/nologin -G nginx nginx \
	&& apk add --no-cache --virtual .build-deps \
		binutils \
		build-base \
		ca-certificates \
		cmake \
		curl \
		gcc \
		g++ \
		gd-dev \
		git \
		gnupg \
		go \
		libc-dev \
		libgcc \
		libstdc++ \
		libxslt-dev \
		linux-headers \
		make \
		pcre-dev \
		perl-dev \
		tar \
		tzdata \
		zlib-dev \
	&& [ $BUILD_GEOIP2 = "false" ] || ( \
		CONFIG="$CONFIG --add-dynamic-module=/usr/src/ngx_http_geoip2_module" && \
		apk add --no-cache gzip libmaxminddb-dev && \
		git clone --depth 1 https://github.com/leev/ngx_http_geoip2_module.git /usr/src/ngx_http_geoip2_module && \
		curl -L "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=$GEOIP2_LICENSE_KEY&suffix=tar.gz" -o GeoLite2-Country.mmdb.gz && \
		gunzip GeoLite2-Country.mmdb.gz && \
		mkdir -p /etc/nginx/GeoIP2 && \
		cp ./GeoLite2-Country.mmdb /etc/nginx/GeoIP2/) \
	&& curl -fSL https://nginx.org/download/nginx-$NGINX_VERSION.tar.gz -o nginx.tar.gz \
	&& curl -fSL https://nginx.org/download/nginx-$NGINX_VERSION.tar.gz.asc  -o nginx.tar.gz.asc \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& found=''; \
	for server in \
		ha.pool.sks-keyservers.net \
		hkp://keyserver.ubuntu.com:80 \
		hkp://p80.pool.sks-keyservers.net:80 \
		pgp.mit.edu \
	; do \
		echo "Fetching GPG key $GPG_KEYS from $server"; \
		gpg --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$GPG_KEYS" && found=yes && break; \
	done; \
	test -z "$found" && echo >&2 "error: failed to fetch GPG key $GPG_KEYS" && exit 1; \
	gpg --batch --verify nginx.tar.gz.asc nginx.tar.gz \
	&& rm -rf "$GNUPGHOME" nginx.tar.gz.asc \
	&& mkdir -p /usr/src \
	\
	&& git clone --depth=1 --recurse-submodules --shallow-submodules https://github.com/google/ngx_brotli /usr/src/ngx_brotli \
	&& git clone --depth=1 https://github.com/openresty/headers-more-nginx-module /usr/src/ngx_headers_more \
	&& (git clone --depth=1 https://boringssl.googlesource.com/boringssl /usr/src/boringssl \
		&& sed -i 's@out \([>=]\) TLS1_2_VERSION@out \1 TLS1_3_VERSION@' /usr/src/boringssl/ssl/ssl_lib.cc \
		&& sed -i 's@ssl->version[ ]*=[ ]*TLS1_2_VERSION@ssl->version = TLS1_3_VERSION@' /usr/src/boringssl/ssl/s3_lib.cc \
		&& sed -i 's@(SSL3_VERSION, TLS1_2_VERSION@(SSL3_VERSION, TLS1_3_VERSION@' /usr/src/boringssl/ssl/ssl_test.cc \
		&& sed -i 's@\$shaext[ ]*=[ ]*0;@\$shaext = 1;@' /usr/src/boringssl/crypto/*/asm/*.pl \
		&& sed -i 's@\$avx[ ]*=[ ]*[0|1];@\$avx = 2;@' /usr/src/boringssl/crypto/*/asm/*.pl \
		&& sed -i 's@\$addx[ ]*=[ ]*0;@\$addx = 1;@' /usr/src/boringssl/crypto/*/asm/*.pl \
		&& mkdir -p /usr/src/boringssl/build /usr/src/boringssl/.openssl/lib /usr/src/boringssl/.openssl/include \
		&& ln -sf /usr/src/boringssl/include/openssl /usr/src/boringssl/.openssl/include/openssl \
		&& touch /usr/src/boringssl/.openssl/include/openssl/ssl.h \
		&& CPPFLAGS='-D_FORTIFY_SOURCE=2' CFLAGS=$CMPL_FLAGS CXXFLAGS=$CMPL_FLAGS cmake -B/usr/src/boringssl/build -H/usr/src/boringssl -DCMAKE_BUILD_TYPE=RelWithDebInfo \
		&& make -C/usr/src/boringssl/build -j$(getconf _NPROCESSORS_ONLN) \
		&& cp /usr/src/boringssl/build/crypto/libcrypto.a /usr/src/boringssl/build/ssl/libssl.a /usr/src/boringssl/.openssl/lib) \
	\
	&& tar -zxC /usr/src -f nginx.tar.gz \
	&& rm nginx.tar.gz \
	&& cd /usr/src/nginx-$NGINX_VERSION \
	&& curl -fSL https://raw.githubusercontent.com/nginx-modules/ngx_http_tls_dyn_size/0.5/nginx__dynamic_tls_records_1.17.7%2B.patch -o dynamic_tls_records.patch \
	&& patch -p1 < dynamic_tls_records.patch \
	&& [ $BUILD_DEBUG = "false" ] || ( true \
	&& ./configure $CONFIG --with-debug --with-cc-opt="-I/usr/src/boringssl/.openssl/include" --with-ld-opt="-L/usr/src/boringssl/.openssl/lib" \
	&& make -j$(getconf _NPROCESSORS_ONLN) \
	&& mv objs/nginx objs/nginx-debug \
	&& mv objs/ngx_http_xslt_filter_module.so objs/ngx_http_xslt_filter_module-debug.so \
	&& mv objs/ngx_http_image_filter_module.so objs/ngx_http_image_filter_module-debug.so \
	&& mv objs/ngx_http_perl_module.so objs/ngx_http_perl_module-debug.so \
	) \
	&& ./configure $CONFIG "$CC_OPTS" "$LD_OPTS" \
	&& make -j$(getconf _NPROCESSORS_ONLN) \
	&& make install \
	&& rm -rf /etc/nginx/html/ \
	&& mkdir /etc/nginx/conf.d/ \
	&& mkdir -p /usr/share/nginx/html/ \
	&& install -m644 html/index.html /usr/share/nginx/html/ \
	&& install -m644 html/50x.html /usr/share/nginx/html/ \
	&& [ $BUILD_DEBUG = "false" ] || ( true \
	&& install -m755 objs/nginx-debug /usr/sbin/nginx-debug \
	&& install -m755 objs/ngx_http_xslt_filter_module-debug.so /usr/lib/nginx/modules/ngx_http_xslt_filter_module-debug.so \
	&& install -m755 objs/ngx_http_image_filter_module-debug.so /usr/lib/nginx/modules/ngx_http_image_filter_module-debug.so \
	&& install -m755 objs/ngx_http_perl_module-debug.so /usr/lib/nginx/modules/ngx_http_perl_module-debug.so \
	) \
	&& ln -s ../../usr/lib/nginx/modules /etc/nginx/modules \
	&& strip /usr/sbin/nginx* \
	&& strip /usr/lib/nginx/modules/*.so \
	&& rm -rf /usr/src/nginx-$NGINX_VERSION \
	&& rm -rf /usr/src/boringssl /usr/src/ngx_* \
	\
	# Bring in gettext so we can get `envsubst`, then throw
	# the rest away. To do this, we need to install `gettext`
	# then move `envsubst` out of the way so `gettext` can
	# be deleted completely, then move `envsubst` back.
	&& apk add --no-cache --virtual .gettext gettext \
	&& mv /usr/bin/envsubst /tmp/ \
	\
	&& runDeps="$( \
		scanelf --needed --nobanner /usr/sbin/nginx /usr/lib/nginx/modules/*.so /tmp/envsubst \
			| awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
			| sort -u \
			| xargs -r apk info --installed \
			| sort -u \
	) tzdata ca-certificates" \
	&& apk add --no-cache --virtual .nginx-rundeps $runDeps \
	&& apk del .build-deps \
	&& apk del .gettext \
	&& mv /tmp/envsubst /usr/local/bin/ \
	\
	# forward request and error logs to docker log collector
	&& ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log \
	&& echo "" && nginx -V;

COPY nginx.conf /etc/nginx/nginx.conf
COPY nginx.vh.no-default.conf /etc/nginx/conf.d/default.conf

LABEL description="NGINX Docker built top of rolling release BoringSSL" \
      maintainer="Denis Denisov <denji0k@gmail.com>" \
      modifiedby="IMBA-TJD" \
      openssl="BoringSSL" \
      nginx="nginx $NGINX_VERSION"

EXPOSE 80 443

STOPSIGNAL SIGTERM

CMD ["nginx", "-g", "daemon off;"]
