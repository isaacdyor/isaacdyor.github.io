---
title: "Create responsive navbar with React and Tailwind using the same markdown"
date: 2023-12-15
permalink: /posts/2023/12/react-navbar/
excerpt: Build a responsive navbar with React and Tailwind CSS using a single markup for both mobile and desktop views. This guide walks you through creating a dynamic, maintainable navbar that adjusts seamlessly to different screen sizes.
---

## Introduction

When building a responsive navbar you need to render two different views for desktop and mobile. One approach is to hardcode the two different views and conditionally render them based on the screen size. Although this method seems simple in the long term, it makes refactoring more complicated because if you want to change one of your links you have to do it in two places. For this reason the better option is to use the same markdown and apply styles conditionally so it works with both desktop and mobile. This way if you want to change something you can do it in one place and the changes will be applied to both views.

## Basic mobile navbar

Tailwind has a mobile first design approach. For this reason I will start by building the mobile navbar, and later I will apply conditional styles to make it work with desktop.

First we are just going to create the flexbox for the navbar with the logo. This eventually will use state for the hamburger menu so it needs to be a client component.

```
"use client";
import Image from "next/image";
import Link from "next/link";
import Logo from "/public/logo.png";

const Navbar: React.FC = () => {
  return (
    <div className="flex items-center justify-between p-3 border-b border-b-border">
      <Link href={"/"} className="shrink-0 px-4">
        <Image src={Logo} alt="Spark Royalty Logo" width={250} height={250} />
      </Link>
    </div>
  );
};

export default Navbar;
```

![Current navbar](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rdj2tpul4zrs5q6vrkw7.png)

Since we are on the mobile view we want to add a hamburger menu to toggle the links visibility. I am using [heroicons](https://heroicons.com/). We use some basic react state to know whether or not the hambuger is open, and we conditionally render either the hamburger or an X.

```
"use client";
import Image from "next/image";
import Link from "next/link";
import Logo from "/public/logo.png";
import { useState } from "react";
import { XMarkIcon, Bars3Icon } from "@heroicons/react/24/solid";

const Navbar: React.FC = () => {
  const [menuOpen, setMenuOpen] = useState(false);

  const toggleMenu = () => {
    setMenuOpen(!menuOpen);
  };

  return (
    <div className="flex items-center justify-between p-3 border-b border-b-border">
      <Link href={"/"} className="shrink-0 px-4">
        <Image src={Logo} alt="Spark Royalty Logo" width={250} height={250} />
      </Link>
      <button onClick={toggleMenu} className="md:hidden">
        {menuOpen ? (
          <XMarkIcon className="h-7 w-7" />
        ) : (
          <Bars3Icon className="h-7 w-7" />
        )}
      </button>
    </div>
  );
};

export default Navbar;
```

![Current navbar](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/55yrqwqclufy5v7pdgko.png)

Right now the hamburger menu isnt actually rendering anything when it is opened, so lets add some links and a sign up and login button. I am using shadcn for my buttons and my colors. Check out my [last post](https://dev.to/isaacdyor/setting-up-nextjs-project-with-prisma-200j) if you want to get that set up.

```
"use client";
import Image from "next/image";
import Link from "next/link";
import Logo from "/public/logo.png";
import { useState } from "react";
import { XMarkIcon, Bars3Icon } from "@heroicons/react/24/solid";
import { Button } from "./ui/button";

const Navbar: React.FC = () => {
  const [menuOpen, setMenuOpen] = useState(false);

  const toggleMenu = () => {
    setMenuOpen(!menuOpen);
  };

  const routes: { title: string; href: string }[] = [
    { title: "Features", href: "#features" },
    { title: "Resources", href: "#resources" },
    { title: "Pricing", href: "#pricing" },
  ];

  return (
    <div className="flex items-center justify-between p-3 border-b border-b-border">
      <Link href={"/"} className="shrink-0 px-4">
        <Image src={Logo} alt="Spark Royalty Logo" width={250} height={250} />
      </Link>
      <div
        className={`flex items-center grow justify-start flex-col absolute top-[71.5px]  right-0 w-full  ${
          menuOpen ? "visible" : "invisible"
        }`}
      >
        <div className="flex flex-col justify-end w-full py-2 px-4 gap-1 bg-background">
          {routes.map((route, index) => (
            <Link
              key={index}
              href={route.href}
              className={`inline-flex h-10 w-full items-center rounded-md px-4 py-2 text-sm font-medium transition-colors hover:bg-accent hover:text-accent-foreground`}
            >
              {route.title}
            </Link>
          ))}

          <Link href={"/login"} className="w-full ">
            <Button variant="secondary" className="w-full">
              Log In
            </Button>
          </Link>
          <Link href="/signup" className="w-full">
            <Button variant="default" className="w-full">
              Sign Up
            </Button>
          </Link>
        </div>
        <div className="h-full w-full bg-background/70" />
      </div>
      <button onClick={toggleMenu}>
        {menuOpen ? (
          <XMarkIcon className="h-7 w-7" />
        ) : (
          <Bars3Icon className="h-7 w-7" />
        )}
      </button>
    </div>
  );
};

export default Navbar;
```

I create a flex column to store the flex column with all of the links as well as a div to take up the rest of the space. This div is opaque to cover up whatever is on the screen currently. Makes it look like this:

![Current navbar](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fber83eqpzybom3yet15.png)

## Making it work for desktop

Now that our navbar works for mobile we want to make it work for desktop. We are going to use the same markup, just give it different styles. We will do this by applying tailwind styles with the md: prefix so they will only be applied on medium screen sizes and larger.

There are a couple of things we need to change:

- Make the content always visible so the hamburger state doesn't affect it
- Make it a flex row instead of flex col
- Make the position static instead of fixed
- Hide the hamburger menu
- Make the width of the buttons and links auto instead of full
- Add some styles specific to desktop, like padding between the login and signup buttons.

```
"use client";
import Image from "next/image";
import Link from "next/link";
import Logo from "/public/logo.png";
import { useState } from "react";
import { XMarkIcon, Bars3Icon } from "@heroicons/react/24/solid";
import { Button } from "./ui/button";

const Navbar: React.FC = () => {
  const [menuOpen, setMenuOpen] = useState(false);

  const toggleMenu = () => {
    setMenuOpen(!menuOpen);
  };

  const routes: { title: string; href: string }[] = [
    { title: "Features", href: "#features" },
    { title: "Resources", href: "#resources" },
    { title: "Pricing", href: "#pricing" },
  ];

  return (
    <div className="flex items-center justify-between p-3 border-b border-b-border">
      <Link href={"/"} className="shrink-0 px-4">
        <Image src={Logo} alt="Spark Royalty Logo" width={250} height={250} />
      </Link>
      <div
        className={`flex items-center h-[calc(100vh-71.5px)] grow justify-start flex-col absolute top-[71.5px]  right-0 w-full  md:visible md:flex-row md:justify-end md:static md:h-auto ${
          menuOpen ? "visible" : "invisible"
        }  `}
      >
        <div className="flex flex-col justify-end w-full py-2 px-4 gap-1 bg-background md:flex-row">
          {routes.map((route, index) => (
            <Link
              key={index}
              href={route.href}
              className={`inline-flex h-10 w-full items-center rounded-md px-4 py-2 text-sm font-medium transition-colors hover:bg-accent hover:text-accent-foreground md:w-auto`}
            >
              {route.title}
            </Link>
          ))}

          <Link href={"/login"} className=" w-full md:px-1 md:w-auto">
            <Button variant="secondary" className="w-full">
              Log In
            </Button>
          </Link>
          <Link href="/signup" className="w-full md:w-auto">
            <Button variant="default" className="w-full">
              Sign Up
            </Button>
          </Link>
        </div>
        <div className="h-full w-full bg-background/70 md:hidden" />
      </div>
      <button onClick={toggleMenu} className="md:hidden">
        {menuOpen ? (
          <XMarkIcon className="h-7 w-7" />
        ) : (
          <Bars3Icon className="h-7 w-7" />
        )}
      </button>
    </div>
  );
};

export default Navbar;
```

![Current navbar](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ecem7z7p6fohljeano3t.png)

Now you have a responsive navbar using the same markdown for each view, so if you want to add a new link or change the text, you only have to do it one place!
w
