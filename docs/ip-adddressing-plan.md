Site	Department		Subnet		Mask	Usable Range	Broadcast
HQ	Sales (20)		10.10.0.0/27	.224	.1–.30		.31
HQ	Guest/Wireless (15)	10.10.0.32/27	.224	.33–.62		.63
HQ	IT/Management (10)	10.10.0.64/28	.240	.65–.78		.79
HQ	Servers (5)		10.10.0.80/29	.248	.81–.86		.87
Branch	Sales (10)		10.20.0.0/28	.240	.1–.14		.15
Branch	IT (5)			10.20.0.16/29	.248	.17–.22		.23




Host bits required depends on number of hosts-
	If the number of hosts need to be conigure is 20, the host bits required is 5 because, for 5 host bits total usable ip's are (2^5 - 2)
	i.e 30, this filled our requirement and let us some more ip'  to use further 
