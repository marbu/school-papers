all: bukatovic.assignment.pdf clean

# remove temporary files
clean:
	rm -rf *.log *.aux *.bbl *.blg *.out

# main document
%.pdf: %.tex literature.bib
	pdflatex $<
	bibtex $(basename $<)
	pdflatex $<
	pdflatex $<
