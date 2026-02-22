+++
title = 'Using libi2c'
date = '2026-02-17T22:00:10-07:00'
+++

I recently worked on a project which used `i2cset` and `i2cget` via
system()[^1] to implement an i2c smbus device driver in C++. I thought it was a
shame because the widely-available libi2c C library (which is actually
developed in the same repository as `i2cset` and `i2cget`) makes manipulating
i2c smbus devices very easy.

Using system() means that if there are any breaking changes to i2c-tools in the
future, the code will encounter a run-time error caused by changes to the
command-line options of `i2cset` and `i2cget`. Comparatively, linking against
libi2c directly means that if there are any breaking changes, it will cause a
compile-time error. As a matter of principle, compile-time errors are
preferable to run-time errors.

As an added benefit, driving i2c devices directly with libi2c incurs a much
smaller performance overhead because...

1. The caller can keep the device file descriptor open between i2c read/write
   commands.
2. Foregoing the system() call rids the code of all sorts of performance hits
   involved with spinning up a new process.

When I tested the performance of the two different approaches, I
found that smbus read commands were _over 25x faster_ when using libi2c
directly.

What follows is some example C code for reading a value from an i2c smbus
device from userspace using libi2c[^2]. Also see the [i2c-tools source
code](https://git.kernel.org/pub/scm/utils/i2c-tools/i2c-tools.git) which is
very readable and contains more detailed usage of this interface. When using
libi2c, consider using the `I2C_FUNCS` ioctl to check at run-time that the i2c
controller driver supports the desired operations before using them. Otherwise,
you may encounter errors with `i2c_smbus_*` functions returning negative values
when a driver doesn't support a particular operation.

```c
#include <errno.h>
#include <fcntl.h>
#include <i2c/smbus.h>
#include <linux/i2c-dev.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(int argc, char **argv) {
	int file;
	int32_t res;
	int status = 0;

	const char *path = "/dev/i2c-0";
	const uint8_t slave = 0x44;
	const uint8_t reg = 0x03;

	file = open(path, O_RDWR);
	if(file < 0) {
		fprintf(stderr, "Couldn't open %s: %s\n", path, strerror(errno));
		status = 1;
		goto cleanup;
	}

	if (ioctl(file, I2C_SLAVE, slave) < 0) {
		fprintf(stderr, "Couldn't set i2c slave address: %s\n", strerror(errno));
		status = 2;
		goto cleanup_file;
	}

	res = i2c_smbus_read_byte_data(file, reg);

	if(res < 0) {
		fprintf(stderr, "Couldn't read i2c register: %s\n", strerror(errno));
		status = 3;
		goto cleanup_file;
	}

    printf("Controller %s\n", path);
    printf("Slave 0x%02x\n", slave);
    printf("Register 0x%02x\n", reg);
    printf("Contained value 0x%02x\n", (uint8_t)res);

cleanup_file:
	close(file);
cleanup:
	return status;
}
```

By the way, you may have heard that `goto` is tautologically bad. In this
particular case, it is perhaps a little overkill, but not inappropriate. Stack
unwinding is a benign use of `goto`.

[^1]: See my thoughts on using system() to implement logic from low-level
    programming languages [here](/notes/exec-code-smell/)
[^2]: See https://docs.kernel.org/i2c/dev-interface.html
