# Document History

1. Does the working group (WG) consensus represent the strong concurrence of a
few individuals, with others being silent, or did it reach broad agreement?

The document represents the consensus of a broad group of participants. The
genesis of the document was the combination of several distinct technical
proposals. Through the lengthy WG process, the design has converged towards the
singular design now presented.

2. Was there controversy about particular points, or were there decisions where
the consensus was particularly rough?

The document followed the Github workflow, used for all QUIC WG documents. Since
the draft was adopted in 2022, almost 300 issues, and over 326 PRs were opened.
Some of the largest discussion items revolved around single vs. multiple packet
spaces and the relationship between connection IDs and path IDs. There was a lot
of interest and energy in such topics, and although they took some time to
overcome, I am confident that the final consensus is not controversial or
particularly rough.

3. Has anyone threatened an appeal or otherwise indicated extreme discontent? If
so, please summarize the areas of conflict in separate email messages to the
responsible Area Director. (It should be in a separate email because this
questionnaire is publicly available.)

I have seen no such threats or indications.

4. For protocol documents, are there existing implementations of the contents of
the document? Have a significant number of potential implementers indicated
plans to implement? Are any existing implementations reported somewhere,
either in the document itself (as RFC 7942 recommends) or elsewhere
(where)?

The authors consist of several groups that have been independently implementing
the draft iterations and carrying out regular interoperability tests (e.g.,
during hackathon events) and reporting results during WG meetings. I am aware of
Alibaba's production deployment of the document, although I believe this is
primarily a self-interop deployment.

Outside the authors, a range of other QUIC implementations have implemented the
document to some level of maturity.

I am aware of other implementation and deployment interest in the features of
the document, primarily driven by the interest in the potential for improved
durability and availability of QUIC connections that can use multiple paths.

3GPP have repeatedly expressed a strong interest in this document.

# Additional Reviews

5. Do the contents of this document closely interact with technologies in other
IETF working groups or external organizations, and would it therefore benefit
from their review? Have those reviews occurred? If yes, describe which
reviews took place.

The document describes an extension to QUIC to enable the simultaneous usage of
multiple paths for a single connection. Members of the QUIC WG are active in
many adjacent groups, and have prior relevant expertise in technologies that are
relevant. Any necessary interactions have likely happened organically through
the course of consensus.

The IETF last call is an excellent time to gather wider input. I do not think
any particular WG or area needs to pay special attention.

As noted in the previous answer, 3GPP have expressed a lot of interest in the
document over the years. My first suggestion would be to use informal means to
make them aware that we intend to send this on to the IESG. We may,
alternatively, consider issuing a formal information liasion but I would welcome
input on that from those closer to the 3GPP work.

6. Describe how the document meets any required formal expert review criteria,
such as the MIB Doctor, YANG Doctor, media type, and URI type reviews.

The document contains no MIB, YANG, media type or URI related contents.

7. If the document contains a YANG module, has the final version of the module
been checked with any of the recommended validation tools for syntax and
formatting validation? If there are any resulting errors or warnings, what is
the justification for not fixing them at this time? Does the YANG module
comply with the Network Management Datastore Architecture (NMDA) as specified
in RFC 8342?

N/A

8. Describe reviews and automated checks performed to validate sections of the
final version of the document written in a formal language, such as XML code,
BNF rules, MIB definitions, CBOR's CDDL, etc.

The document contains no relevant formal language sections.

# Document Shepherd Checks

9. Based on the shepherd's review of the document, is it their opinion that this
document is needed, clearly written, complete, correctly designed, and ready
to be handed off to the responsible Area Director?

Yes,

10. Several IETF Areas have assembled lists of common issues that their
reviewers encounter. For which areas have such issues been identified
and addressed? For which does this still need to happen in subsequent
reviews?

To my knowledge, there have been no issues identified. Therefore, I expect IETF
last call to catch anything relevant.

11. What type of RFC publication is being requested on the IETF stream (Best
Current Practice, Proposed Standard, Internet Standard,
Informational, Experimental or Historic)? Why is this the proper type
of RFC? Do all Datatracker state attributes correctly reflect this intent?

The intended status is Proposed Standard. This is a proper type for the QUIC
extension elements defined in the document. Care was taken throughout the
consensus process to ensure research topics were kept out of scope, meaning the
an experimental status for the protocol itself is not required.

12. Have reasonable efforts been made to remind all authors of the intellectual
property rights (IPR) disclosure obligations described in BCP 79? To
the best of your knowledge, have all required disclosures been filed? If
not, explain why. If yes, summarize any relevant discussion, including links
to publicly-available messages when applicable.

TBC: each author responded

13. Has each author, editor, and contributor shown their willingness to be
listed as such? If the total number of authors and editors on the front page
is greater than five, please provide a justification.

TBC: each author responded

The number of authors on the front page is 6. This was discussed extensively in
the lead up to adoption with the author group, chairs and the responsible AD at
the time. This number is justified by the document being an amalgamation of
various design proposals that were unified via hard work, determination and
compromise.

14. Document any remaining I-D nits in this document. Simply running the idnits
tool is not enough; please review the "Content Guidelines" on
authors.ietf.org. (Also note that the current idnits tool generates
some incorrect warnings; a rewrite is underway.)

I have created https://github.com/quicwg/multipath/issues/620. Beyond that,
there are no other issues to my knowledge.

15. Should any informative references be normative or vice-versa? See the IESG
Statement on Normative and Informative References.

All references appear to be the correct type.

16. List any normative references that are not freely available to anyone. Did
the community have sufficient access to review any such normative
references?

N/A, all normative references are IETF RFCs.

17. Are there any normative downward references (see RFC 3967 and BCP
97) that are not already listed in the DOWNREF registry? If so,
list them.

No

18. Are there normative references to documents that are not ready to be
submitted to the IESG for publication or are otherwise in an unclear state?
If so, what is the plan for their completion?

No

19. Will publication of this document change the status of any existing RFCs? If
so, does the Datatracker metadata correctly reflect this and are those RFCs
listed on the title page, in the abstract, and discussed in the
introduction? If not, explain why and point to the part of the document
where the relationship of this document to these other RFCs is discussed.

No

20. Describe the document shepherd's review of the IANA considerations section,
especially with regard to its consistency with the body of the document.
Confirm that all aspects of the document requiring IANA assignments are
associated with the appropriate reservations in IANA registries. Confirm
that any referenced IANA registries have been clearly identified. Confirm
that each newly created IANA registry specifies its initial contents,
allocations procedures, and a reasonable name (see RFC 8126).

There are no new IANA registries being defined.

Several new elements are being registered in existing QUIC registries. The
registration requests are compliant with the guidance defined in RFC 9000. Some
nits were identified and reported on
https://github.com/quicwg/multipath/issues/621.

As is common for QUIC extensions, prior to publication a set of provisional
codepoints have been reserved. The IANA considerations section contains clear
instructions to IANA requesting the allocation of shorter codepoints once the
draft has been approved for publication.

21. List any new IANA registries that require Designated Expert Review for
future allocations. Are the instructions to the Designated Expert clear?
Please include suggestions of designated experts, if appropriate.

There are no new IANA registries being defined.