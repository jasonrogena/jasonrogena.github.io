---
layout: post_page
title: Simple Localization on Android
comments: true
---

Localizing apps to languages that are officially supported in Android is a bit straightforward. However, for unsupported languages, it might be a bit tricky especially if there are special characters involved.

I needed to translate my app to a bunch of local Kenyan languages. Adding support for these languages proved very easy (adding the translated strings to the strings.xml resource file). Mainly because all of these languages use the English alphabet. It was however the code that I ended up being concerned about. All references to string resources in the app's Java code were enclosed in long condition code block checking the user's current language:

    String translatedString = "";
    switch(userLang){
        case "english": translatedString = context.getString("R.string.string_in_english");
        case "kiswahili": translatedString = context.getString("R.string.string_in_kiswahili");
        case "kalenjin": translatedString = context.getString("R.string.string_in_kalenjin");
    }

You'd imagine that translating the app to a new language would be hell, and it was. I had to add the new language to each and every of these condition code blocks (of course forgetting a few here and there). It was simply unsustainable. I changed how the app handles string resources.


### Android String Resources?

You might already know that you can define your strings in xml files then use these strings directly in your layout files or import the strings in your Java code. I'll try to explain how handles the string so that you are able to use them in your Java code.

During the build proccess, a class called **R$string** is generated. **R$string** contains all the string resources. In this class all the strings are stripped of their names that are replaced with hexadecimal IDs. Another class **R** is also generated. This class associates the string names defined in the resource files with the dynamically generated hexadecimal IDs assigned to the strings in **R$string**.

For instance if we had a strings.xml resource file that looks like this:

    <string name="app_name">LML</string>
    <string name="string_0">Boom boom</string>
    <string name="string_1">Pow pow</string>

Our **R** would look like this:

    public final class R {
        public static final class string{
           public static final int app_name = 0x7f080023;
           public static final int string_0 = 0x7f080024;
           public static final int string_1 = 0x7f080025;
        }
    }

When referencing a string in the Java code, doing something like this:

    context.getString(R.string.string_0);

will in essence:

 1. Get the hexadecimal ID for the string that has the name 'string_0' from the **R** class.
 2. Pass this hexadecimal ID on to the getString method that will search **R$string** and return the string that corresponds to the given hexadecimal ID.


### The Hack

The hack involved quite a bit of code refactoring. However, when I got things up and running again, the lines of code ended up being less. Here goes how I did it:


#### 1. Language Codes

I came up with unique two character codes for each of the native languages my app supports. For instance:

 - **sw**: Kiswahili
 - **kl**: Kalenjin
 - **kb**: Kikabrasi

If your app is also translated to locales that are officially supported on Android, I suggest you pick codes that do not conflict with any of the two letter ISO codes for the supported languages.


#### 2. Resource Files

I then created a string resource file for each of the native languages my app was translated to. The naming convention I used for the resource files is **string_[two letter language code].xml** e.g:

 - strings_sw.xml
 - strings_kl.xml

I reserved the main string resource file (strings.xml) for general strings such as the app name.


#### 3. Translated Strings

I also adopted a similar naming convention for strings in the native language resource files. For instance, strings inside the **strings_sw.xml** file all had the suffix **_sw** appended to their names e.g:

    <string name="string_0_sw">Boom boom</string>

I also had to make sure that all the native languge resource files had the same set of strings i.e if strings_sw.xml had string_0_sw and string_1_sw then strings_kl.xml should also have string_0_kl and string_1_kl.


#### 4. Translation Class

On the Java end, I had to create a helper class that would help in picking the correct string depending on the user's language. In this class, the method specifically responsible for returning strings looks like this:

    public static String getStringInLocale(String stringName, Context context) {
        String localeCode = getLocaleCode(context); //get the user's preferred language
        String name = stringName+"_"+localeCode;
        String value = null;
        try {
            Field field = R.string.class.getDeclaredField(name);
            int id = field.getInt(field);
            if(id != 0) {
                value = context.getString(id);
            }
            else {
                Log.e(TAG,"no field in class R.string with the name "+name);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return value;
    }

What you pass to this method (as an argument) is the name of the string without the two letter language suffix e.g **string_0**. The method then has to figure out the user's prefered language. It then constructs the string name, as is in the resource files, e.g **string_0_sw**. The next step involves getting the hexadecimal ID for the string from the **R** class. It however does it using the [Field](http://docs.oracle.com/javase/7/docs/api/java/lang/reflect/Field.html) class. The class also has a method for getting string arrays. You can take a took at [it on GitHub](https://github.com/jasonrogena/ngombe_planner-android/blob/master/NgombePlanner/src/main/java/org/cgiar/ilri/np/farmer/backend/Locale.java), if you want to.
