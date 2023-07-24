# Multicast

It's a RSA challenge where we get the ciphertext two times, each time with different exponent.

To solve it, we can use RSA Common Modulus attack

https://blog.0daylabs.com/2015/01/17/rsa-common-modulus-attack-extended-euclidean-algorithm/

```py
import gmpy2
from Crypto.Util.number import long_to_bytes
import re

class RSAModuli:
    def __init__(self):
        self.a = 0
        self.b = 0
        self.m = 0
        self.i = 0

    def gcd(self, num1, num2):
        """
        This function os used to find the GCD of 2 numbers.
        :param num1:
        :param num2:
        :return:
        """
        if num1 < num2:
            num1, num2 = num2, num1
        while num2 != 0:
            num1, num2 = num2, num1 % num2
        return num1

    def extended_euclidean(self, e1, e2):
        """
        The value a is the modular multiplicative inverse of e1 and e2.
        b is calculated from the eqn: (e1*a) + (e2*b) = gcd(e1, e2)
        :param e1: exponent 1
        :param e2: exponent 2
        """
        self.a = gmpy2.invert(e1, e2)
        self.b = (float(self.gcd(e1, e2)-(self.a*e1)))/float(e2)

    def modular_inverse(self, c1, c2, N):
        """
        i is the modular multiplicative inverse of c2 and N.
        i^-b is equal to c2^b. So if the value of b is -ve, we
        have to find out i and then do i^-b.
        Final plain text is given by m = (c1^a) * (i^-b) %N
        :param c1: cipher text 1
        :param c2: cipher text 2
        :param N: Modulus
        """
        i = gmpy2.invert(c2, N)
        mx = pow(c1, self.a, N)
        my = pow(i, int(-self.b), N)
        self.m = mx * my % N

    def print_value(self):
        print(re.findall(br'ecsc23\{.+\}', long_to_bytes(self.m))[0])


def main():
    c = RSAModuli()
    f = open('data.txt').read().splitlines()
    N, e1, e2, c1, c2 = [int(x[4:]) for x in f]
    c.extended_euclidean(e1, e2)
    c.modular_inverse(c1, c2, N)
    c.print_value()

if __name__ == '__main__':
    main()

```