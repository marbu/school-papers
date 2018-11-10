# http://docutils.sourceforge.net
RST2HTML=rst2html
RST2XML=rst2xml
RST2LATEX=rst2latex
PDFLATEX=pdflatex

all: paper clean_

slides:  ipv6talk.pdf
paper:   ipv6paper.html
tarball: ipv6paper.tar.gz

clean:
	-rm ipv6talk.pdf ipv6paper.html ipv6paper.tex ipv6paper.xml ipv6paper.pdf ipv6paper.tar.gz

clean_:
	-rm *.aux *.nav *.log *.out *.snm *.toc *.vrb

# talk

ipv6talk.pdf: ipv6talk.tex
	pdflatex $^
	pdflatex $^
	pdflatex $^

# paper

ipv6paper.tar.gz: ipv6paper.rst Makefile ipv6paper.html
	tar cfz $@ $^

stat:
	-wc ipv6paper.rst

paperpush: ipv6paper.html ipv6paper.tar.gz
	scp $^ aisa:~/public_html/atol

# docutils: general rules

%.html: %.rst
	$(RST2HTML) $^ > $@

%.tex: %.rst
	$(RST2LATEX) --output-encoding=utf8 $^ > $@

%.xml: %.rst
	$(RST2XML) $^ > $@

%.pdf: %.tex
	$(PDFLATEX) $^
	-rm *.out *.log *.aux
