# Beautifulsoup-exercise
import requests
import pandas as pd
from bs4 import BeautifulSoup
import re
from tkinter import *
import tkinter as tk
from tkinter import filedialog
from tkinter import messagebox
import webbrowser
import os
import xlsxwriter
import time
from time import sleep
# here is the function to obtain the destination for the spreedsheet
def get_destination():
    destination = filedialog.askdirectory()
    dir_destination.configure(text=destination)
    dir_destination.delete(0, 'end')
    return dir_destination.insert(0,destination)

# function to extract the website data, parse and create the dataframe output
def analyzer():
    root_site = r'https://en.wikipedia.org/wiki/'
    # page = requests.get('https://en.wikipedia.org/wiki/Natural_language_processing')
    # soup = BeautifulSoup(page.content, 'html.parser')
    destination = dir_destination.get()
    website = website_origin.get()
    page_i = requests.get(website)
    soup_i = BeautifulSoup(page_i.content, 'html.parser')
    writer = pd.ExcelWriter(rf'{destination}\output.xlsx', engine='xlsxwriter')
    for element in soup_i.find_all('dt'):
        name = str(element.a.get('href')).split(r'/')[-1]
        page = requests.get(root_site + name)
        print(name)
        time.sleep(3)
        soup = BeautifulSoup(page.content, 'html.parser')
        # in order to create dataframe, i create lists of dictionaries
        authors = []
        release_year = []
        title = []
        isbn = []
        # obtaining the elements in the referece
        for ref in soup.find_all('span', {'class':'reference-text'}):
            # creation of a BeautifulSoup object for data parsing
            ref_b = BeautifulSoup(str(ref), 'html.parser')
            for tag in ref_b.findAll(True,{'id':True}):
                if re.search("\"(.*?)\"", tag.text) is None:
                    title.append({'title': tag.i.contents[0] if tag.i is not None else re.search("\"(.*?)\"", tag.text).group(0)})
                elif re.search("\"(.*?)\"", tag.text) is not None:
                    title.append({'title': re.search("\"(.*?)\"", tag.text).group(0)})
                else:
                    title.append({'title': tag.i.contents[0]})
                # in this section, the extraction of the ISBN
                if re.search("ISBN", tag.text) is not None:
                    isbn.append({'ISBN' : str(tag.text).split('ISBN')[-1].replace(u'\xa0', '')})
                elif re.search("ISBN", tag.text) is None:
                    isbn.append({'ISBN' : ''})
                # to obtain the authors the process was easier
                authors.append({'authors' : re.split('[("]',tag.text)[0]})
                # to extract the year I focused in the id element
                # release_year.append({'release_year' : str(tag['id'][-4:]) if str(tag['id'][-4:]).isnumeric() else None}) #str(tag.i.text).split(' ')[1]
                if str(tag['id'][-4:]).isnumeric():
                    release_year.append({'release_year' : str(tag['id'][-4:])})
                else:
                    try:
                        release_year.append({'release_year' : str(tag.i.text).split(' ')[1] if str(str(tag.i.text).split(' ')[1]).isnumeric() else None})
                    except:
                        release_year.append({'release_year' : element for element in [number for number in re.findall(r'.*([1-2][0-9]{3})', str(tag)) if int(number) > 1100]})

        # I create one DataFrame object for each list of dictionaries for later concatenation
        df_authors = pd.DataFrame(authors)
        df_release_year = pd.DataFrame(release_year)
        df_title = pd.DataFrame(title)
        df_isbn = pd.DataFrame(isbn)
        result = pd.concat([df_authors, df_release_year, df_title, df_isbn], axis=1, sort=False)
        # last data cleaning previous to the termination
        result.replace(['\'', '\"'], '', regex=True, inplace=True)
        # saving the dataframe in the destination choosed by the user
        result.to_excel(writer, sheet_name=name[:30], index=False)
    return writer.save(), tk.messagebox.showinfo(title='Warning', message='Check your folder!')
# I created a gui for better experience, I hope you enjoy
root = tk.Tk(className='Wikipage References Checker')

root.resizable(width=True, height=True)
# title object
title = tk.Label(master=root, text="Task Pathfinder")
title.config(font=("Amazon Ember", 25))
title.grid(row=1, columnspan = 3)
# website extraction field object
website_origin_label = tk.Label(master=root, text="Choose your site: ")
website_origin_label.config(font=("Amazon Ember", 9))
website_origin_label.grid(row=2, column=0, sticky = W)
website_origin = tk.Entry(master=root, width=50)
website_origin.insert(END, r'https://en.wikipedia.org/wiki/Natural_language_processing')
website_origin.grid(row=2, column=1)
# directory browse object
dir_destination_label = tk.Label(master=root, text="Choose the destination: ")
dir_destination_label.config(font=("Amazon Ember", 9))
dir_destination_label.grid(row=4, column=0, sticky = W)
dir_destination = tk.Entry(master=root, width=50)
dir_destination.grid(row=4, column=1)
dir_destination_getter = tk.Button(master=root, text="Choose path", command=get_destination)
dir_destination_getter.grid(row=4, column=2, sticky='nesw')
# button to start the function
file_creator = tk.Button(master=root, text="Analyze", command=analyzer, bg='orange')
file_creator.grid(row=6, column=1)

tk.mainloop()
