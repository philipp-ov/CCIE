# IP Design
- Four billion (232) IP addresses in IPv4 - keep track of them is incredibly complex, and require a lot of resources, that's why we use prefixes.
- Sticking to prefixes reduces that number down to about one million instead.
- Contiguous ranges of IP addresses are expressed as IP prefixes  
- IP prefixes: An IP prefix is a range of IP addresses, bundled together in powers of two: In the IPv4 space, two addresses form a /31 prefix, four form a /30, and so on, all the way up to /0, which is shorthand for “all IPv4 prefixes''. The same applies for IPv6  but instead of aggregating 32 bits at most, you can aggregate up to 128 bits. The figure below shows this relationship between IP prefixes, in reverse -- a /24 contains two /25s that contains two /26s, and so on.
