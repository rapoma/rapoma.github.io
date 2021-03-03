---
title: Basic implementation of an Elliptic Curve in C++
author: Rapoma
date: 2021-03-03 17:14:00 +0100
categories: [Math, EllipticCurve]
tags: [Elliptic curve, C++, givaro, nathematics]
math: true
---

> A short anecdote, I once participated in a cryptography challenge, in which most of the questions were breaking [ECDH](https://wiki.openssl.org/index.php/Elliptic_Curve_Diffie_Hellman). There are many tools out there but my problem is that instead of using it, I ended up learning to use it. What I mean is that, let say I want to factorize a number and the tool uses brute force to find factor but if I already knew some sets that wont divide the number, of course I would skip them but the tools did not give me that freedom.  



## Introduction

In this post, I am going to share a *very basic* implementation of an Elliptic Curve over a finite field in [_C++_](https://en.wikipedia.org/wiki/C%2B%2B). Using a library for arithmetic and algebraic computation [Givaro](https://casys.gricad-pages.univ-grenoble-alpes.fr/givaro/), this is one of the back-end of [Sagemath](https://www.sagemath.org/). 

I consider the reduced Weierstrass form (field I am going to use is of characteristic different from 2 and 3).

$$y^2 =x^3+Ax+ B$$

To follow along, you might need a basic understanding of ```C++``` and how to use makefile, also I assume that you know what an Elliptic Curve is. In this post I consider an elliptic curve over a finite field which is actually a group. For not going so deep, just consider it as a set of rational point. Also I am going to use affine coordinates representation, of course one can tweak the code to use projective coordinates depending on what is your need, in here lets just keep it simple.

## Steps

### Getting started

To install Givaro, you could use the instruction [here](https://casys.gricad-pages.univ-grenoble-alpes.fr/givaro/givaro_install.html), this is the first requirement. To confirm the installation, grab one of the examples in the source code.

Take a 10 second to breath 😎. What do we need? good question.

- Curve : Field and coefficients $A$ and $B$.
- Point : Only the element
We then implement the operation on the Elliptic Curve.

- Few theory to refresh our memory about the group operation. $$K$$ be a field of char $$\neq 2,3$$

	Let $$P(x_1,y_1)$$, $$Q(x_2,y_2)$$ points of $$E(\mathbb{K})$$ and $$\mathcal{O}$$ the identity.

	+ Inverse of $$P$$ is $$-P$$ of coordinate $$(x_1, -y_1)$$
	+ Addition 

	case 1: $$P = -Q$$ ==> $$\mathcal{O}$$

	case 2: $$P = Q$$ 
		
	the slope of the tangent line at $$P$$ is $$m = \frac{3x_1^2 + A}{2y_1}$$

	so if $$R(x_3,y_3) = P + Q$$ then we have 

	$$x_3 = m^2 - 2x_1$$

	$$y_3 = m(x_1 - x_3) - y_1$$

	case 3: $$P \neq Q$$

	the slope of the line pass through $$PQ$$ is $$m = \frac{y_2 - y_1}{x_2 - x_1}$$

	so if $$R(x_3,y_3) = P + Q$$ then we have 

	$$x_3 = m^2 - (x_1 + x_2) $$

	$$y_3 = m(x_1 - x_3) - y_1$$
	
### Setup

I organize the development setup as follow.

Choose any directory to work in, mine I call it ```Ecc```, inside make two directories ```include``` and ```src```, and a ```Makefile``` to make our life easy 😎 and code like a boss. 

In the first place, I create a common header, name it ```common.h``` inside ```include```. Here is the content

```c++
#include <stdlib.h>
#include <givaro/modular-integer.h>

using namespace Givaro;

typedef Modular<Integer> ZP;
typedef ZP::Element Element;
```

Leave it like this for now, we can update it later on if needed. Of course, what are they? we are going to use *modular-integer*, for further information, please see the Givaro [documentation](https://casys.gricad-pages.univ-grenoble-alpes.fr/givaro/). The new type are just the *Integer Modulo* $$Z_n$$ and its element. 

Now we are ready to code the project. 

### Point
Implementation of rational point (affine) ```Point.h```.

```c++
#ifndef POINT_H_
#define POINT_H_

class Point
{
private:
    
    bool identity;
    Element x,y;
public:

    Point() : identity(true) {}
    Point(bool b);
    Point(Element x, Element y);
    Element getX() const{
        return this->x;
    };
    void setX(Element _x){
        this->x = _x;
    }
    Element getY() const{
        return this->y;
    };

    void setY(Element _y) {
        this->y = _y;
    };

    bool isIdentity() const{
        return this->identity;
    };

    void setIdentity(bool _identity){
        this->identity = _identity;
    };

    Point operator=(const Point& P);
    bool operator==(const Point& P) const;
    void print();
};

#endif
```

```Point.cpp``` inside ```src```

```c++
#include "common.h"
#include "Point.h"

Point::Point(bool b) {
	this->identity = b;
}


Point::Point(Element x, Element y):identity(false) {
	this->x = x;
	this->y = y;
}


Point Point::operator =(const Point& P) {
	if(P.identity)
	    {
	    this->identity = true;
	    return *this;
	    }

	    this->identity=P.identity;
	    this->x = P.x;
	    this->y = P.y;
	    return *this;
}


bool Point::operator ==(const Point& P) const {
	return (this->identity && P.identity) ||
	                        (!this->identity && !P.identity && this->x == P.x && this->y == P.y);
}


void Point::print() {
	this->identity ? std::cout << "infinity "<< std::endl :std::cout << "[" << this->x << ","
	<< this->y << "]" << std::endl;
}
```

### Curve

Representation of the curve, ```Curve.h```

```c++
#ifndef CURVE_H_
#define CURVE_H_
class Curve
    {
    private:
        ZP* FField;
        Element A,B;

    public:
        Curve();
        Curve(Integer primeField, Integer A, Integer B);
        ~Curve();
        bool isZeroDiscriminant();
        ZP *getField()
            {
            return this->FField;
            };

        Element getA(){
            return this->A;
        };
        Element getB(){
            return this->B;
        };
        void print(); 
    };

#endif
```

```Curve.cpp``` inside ```src```

```c++
#include "common.h"
#include "Curve.h"

Curve::Curve() {
	Integer p;
	do {
        std::cout<< "Enter the Prime modulo p :";
        std::cin>>p;
        this->FField = new ZP(p);
        std::cout << "Enter A : ";
        this->FField->read(std::cin,this->A);
        std::cout << "Enter B : ";
        this->FField->read(std::cin,this->B);
        std::cout<< std::endl;
        }
    while (this->isZeroDiscriminant());
	}


Curve::~Curve() {
	free(FField);
}

Curve::Curve(Integer primeField, Integer A, Integer B)
    {
    this->FField = new ZP(primeField);
	this->FField->init(this->A,A);
	this->FField->init(this->B,B); 

    if (this->isZeroDiscriminant()){
        std::cerr << "[!] Curve not defined, disriminant is Zero" << std::endl;
        std::cout << "[+ INFO ] Field : Z/" << primeField << "Z" << std::endl;
        std::cout << "[+ INFO ] A : " << A << " ; B : " << B << std::endl; 
        abort();
        }
    }
void Curve::print() {
    std::cout << " Field : Z/" << this->FField->residu() << "Z" << std::endl;
    std::cout << " A :" ;
    this->FField->write(std::cout,this->A);
    std::cout << ", B :";
    this->FField->write(std::cout,this->B);
    std::cout<< std::endl;
}

bool Curve::isZeroDiscriminant(){
    Element Acube, Bsquare;
    this->FField->mul(Acube, this->A, this->A);
    this->FField->mulin(Acube, this->A);
    this->FField->mulin(Acube, Integer("4"));

    this->FField->mul(Bsquare, this->B, this->B);
    this->FField->mulin(Bsquare, Integer("27"));

    this->FField->addin(Acube, Bsquare);
    return this->FField->isZero(Acube);
}
```

I just thought that its funnier to have the check of non-singularity in here.

### Elliptic Curve
Elliptic Curve ```EllipticCurve.h``` 

```c++
#ifndef ELLIPTICCURVE_H_
#define ELLIPTICCURVE_H_
class EllipticCurve
{
private:
    typedef ZP::Element Element;
    Curve *curve;
    ZP *FField;
    Point identity;
public:
    EllipticCurve(Curve *c);
    EllipticCurve(Integer primeField, Integer A, Integer B);
    ~EllipticCurve();

    const Point& _inv(Point& Q, const Point& P);
    bool _isInv(const Point& Q, const Point& P);
    Point& _double(Point& R, const Point& P);
    Point& _add(Point&R, const Point &P, const Point& Q);
    Point& _scalar(Point& R, const Point& P,Integer k);

    bool verifyPoint(const Point& P) const;
    void print();

};
#endif
```

 ```EllipticCurve.cpp``` inside ```src```

```c++
#include "common.h"
#include "Point.h"
#include "Curve.h"
#include "EllipticCurve.h"

EllipticCurve::EllipticCurve(Integer primeField, Integer A, Integer B)
    {
	this->curve = new Curve(primeField, A, B);
	this->FField = this->curve->getField();
    this->identity.setIdentity(true);
    }


EllipticCurve::EllipticCurve(Curve* curve) {
    this->curve = curve;
    this->FField = this->curve->getField();
    this->identity.setIdentity(true);
	}

EllipticCurve::~EllipticCurve() {
    free(curve);
    free(FField);
	}


const Point& EllipticCurve::_inv(Point& Q, const Point& P) {
	if(P.isIdentity())
	{
		Q.setIdentity(true);
		return Q;
	}
	else
	{
		Q.setIdentity(false);
		Q.setX(P.getX());
        Element tmp;
		this->FField->sub(tmp,this->FField->zero,P.getY());
        Q.setY(tmp);
		return Q;
	}
	}



Point& EllipticCurve::_double(Point& R, const Point& P) {
	Point tmp;
	this->_inv(tmp,P);
	if(P.isIdentity() || (tmp == P))
	    {
	    R.setIdentity(true);
	    return R;
	    }
	else
		{
		//
		R.setIdentity(false);
		Element tmp;
		Element xs; // 3*x^2
		this->FField->mul(xs,P.getX(),P.getX());
		this->FField->init(tmp,(uint64_t)3);
        this->FField->mulin(xs,tmp);
        //
		Element ty; // 2*y
        this->FField->init(tmp,(uint64_t)2);
        this->FField->mul(ty,P.getY(),tmp);
		//
		Element slope; // m
        this->FField->add(tmp,xs,this->curve->getA());
        this->FField->div(slope,tmp,ty);
		//
		Element slope2; // m^2
        this->FField->mul(slope2,slope,slope);
		//
		Element tx; // 2x
		this->FField->add(tx,P.getX(),P.getX());
        //
		Element x3; // x_3
        this->FField->sub(x3,slope2,tx);
		//
		Element y3; // y_3
        this->FField->sub(tmp,P.getX(),x3);
        this->FField->sub(y3,this->FField->mulin(tmp, slope),P.getY());

        R.setX(x3);
        R.setY(y3);
        return R;
		}
	}


Point& EllipticCurve::_add(Point& R, const Point& P, const Point& Q) {
	Point tmp1, tmp2;
	this->_inv(tmp1, P);
	this->_inv(tmp2, Q);
	if((P.isIdentity() && Q.isIdentity()) || (tmp1 == Q) || (tmp2 == P))
		{
	    R.setIdentity(true);
	    return R;
	    }
	if (P.isIdentity())
	    {
	    R = Q;
	    return R;
	    }
	if (Q.isIdentity())
	    {
	    R = P;
	    return R;
	    }
	if(P==Q)
	   {
		return this->_double(R,P);
	   }
	//
    R.setIdentity(false);
	Element tmp;
	Element num; // y2 - y1
	this->FField->sub(num,Q.getY(),P.getY());
	//
	Element den; // x2 - x1
	this->FField->sub(den,Q.getX(),P.getX());
	//
	Element slope; // m
	this->FField->div(slope,num,den);
	//
	Element slope2; // m^2
	this->FField->mul(slope2,slope,slope);
	// 
	Element x3; // x_3
	this->FField->sub(x3,slope2,this->FField->add(tmp,Q.getX(),P.getX()));
	//
	Element diffx3; // x_1 - x_3
	this->FField->sub(diffx3,P.getX(),x3);
	//
	Element y3; // y_3
	this->FField->mul(tmp,slope,diffx3);
	this->FField->sub(y3,tmp,P.getY());
	
    R.setX(x3);
    R.setY(y3);
    return R;
	}


Point& EllipticCurve::_scalar(Point& R, const Point& P, Integer k) {
	if(P.isIdentity())
	{
		R.setIdentity(true);
		return R;
	}
	Point tmp1, tmp2;
	R.setIdentity(true);
	Point PP = P;
	while(k > 0)
	    {
		if (k % 2 == 1)
			{
			this->_add(tmp1,R,PP);
			R = tmp1;
			}
		tmp2 = this->_double(tmp2,PP);
		PP = tmp2;
		k >>= 1;
		}
	return R;
	}


bool EllipticCurve::verifyPoint(const Point& P) const {
	if(P.isIdentity())
	{
		return true;
	}
	Element x3,y2;
	Element Ax,rhs;
	this->FField->mul(x3,P.getX(),P.getX());
    this->FField->mulin(x3,P.getX());
	this->FField->mul(Ax,P.getX(),this->curve->getA());
    
    this->FField->add(rhs,x3,Ax);
    this->FField->addin(rhs,this->curve->getB());

   
    this->FField->mul(y2,P.getY(),P.getY());
    return y2==rhs;
}


void EllipticCurve::print() {
	std::cout<<"Elliptic Curve Defined by ";
	std::cout<<"y^2 = x^3 + ";
	this->FField->write(std::cout,this->curve->getA());
	std::cout<<"x + ";
	this->FField->write(std::cout,this->curve->getB());
	std::cout<<std::endl;
	//std::cout << this->FField->Modular_implem() << std::endl;
}
```

Now we are ready to test the code. create another file where we add the main, I call mine ```test.cpp``` with the following content

```c++
#include <iostream>

#include "common.h"
#include "Point.h"
#include "Curve.h"
#include "EllipticCurve.h"

using namespace Givaro;

int main(int argc, char** argv)
    {
    /**
    ** y^2 = x^3 + 7 
    **/
    EllipticCurve ecc(Integer("115792089237316195423570985008687907853269984665640564039457584007908834671663"),Integer("0"),Integer("7"));

    ecc.print();

    Point P(Integer("65485170049033141755572552932091555440395722142984193594072873483228125624049"), Integer("73163377763031141032501259779738441094247887834941211187427503803434828368457"));

    if (ecc.verifyPoint(P)){
        P.print();
        std::cout << "Point : verified!!!!" << std::endl;
    };

    Point P2;
    ecc._inv(P2,P);
    if (ecc.verifyPoint(P2)){
        P2.print();
        std::cout << "Inverse : verified" << std::endl;
    };

    Point R; 
    R = ecc._double(R, P);
    if (ecc.verifyPoint(R)){
        R.print();
        std::cout << "double : verified" << std::endl;
    };

    Point Rp ;
    Rp = ecc._add(Rp, P , P);
    if (ecc.verifyPoint(Rp)){
        Rp.print();
        std::cout << "add : verified" << std::endl;
    };

    assert(Rp == R);

    Point R5;
    R5 = ecc._scalar(R5,P,Integer("13257"));
    if (ecc.verifyPoint(R5)){
        R5.print();
        std::cout << "scalar : verified" << std::endl;
    };

    return 0;
    }
```

One last thing, let's create a simple makefile. This is totally optional, I just want to have a clean structure and easy workflow.

```Makefile
######## please change me ########
GMPROOT   := $(HOME)/Developements/homebrew/Cellar/gmp/6.2.1
GIVAROROOT:= $(HOME)/Developements/Givaro
##################################

LDFLAGS   := -L$(GMPROOT)/lib -L$(GIVAROROOT)/lib
INCLUDES  := -I$(GMPROOT)/include -I$(GIVAROROOT)/include -Iinclude
LIBS 	  := -lgmp -lgivaro 

######## here you can use g++ ####
CXX       := clang++
CXXFLAGS  := -std=c++11 -O0 -g3 -Wall -fmessage-length=0


OBJ_DIR  := ./obj
BIN_DIR  := ./bin
TARGET   := main
SRC      := $(wildcard src/*.cpp)

OBJECTS  	 := $(SRC:%.cpp=$(OBJ_DIR)/%.o)
DEPENDENCIES := $(OBJECTS:.o=.d)

all: clean build $(BIN_DIR)/$(TARGET)

$(OBJ_DIR)/%.o: %.cpp
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) $(INCLUDES) -c $< -MMD -o $@
	

$(BIN_DIR)/$(TARGET): $(OBJECTS)
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) -o $(BIN_DIR)/$(TARGET) $^ $(LDFLAGS) $(LIBS)

-include $(DEPENDENCIES)

.PHONY: all build clean run

build:
	@echo "[*] 🛠   Building           "
	@echo 
	@mkdir -p $(OBJ_DIR)
	@mkdir -p $(OBJ_DIR)

clean:
	@echo "[*] 🧻🧻 Cleaning           "
	@echo
	-@rm -rvf $(OBJ_DIR)/*
	-@rm -rvf $(BIN_DIR)/*

run: all
	@echo "[*] 🚀 Start Running       "
	@echo 
	$(BIN_DIR)/$(TARGET)
	@echo
	@echo "[*] Finish running         "
```

Now go to you terminal and hit ```make run``` you should see a very cool picture of yourself 😂. Well it should work smoothly. In case it doesnt, join me in the comment section. Also you can always check the [gitrepo](https://github.com/rapoma/ecc-basis).

## Tips

1. Try more example for testing, use other tool to verify the result. 
2. If you have any suggestion and need help to make it, just use the comment section.
3. Next step? I do not know yet.





